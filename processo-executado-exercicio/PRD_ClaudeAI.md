# PRD: Plataforma de Pedidos — Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-06-27
Responsável: Larissa (Tech Lead)

---

### Resumo

O sistema de webhooks permite que clientes B2B sejam notificados automaticamente quando o status de seus pedidos muda na plataforma, eliminando a necessidade de polling manual via `GET /orders`. Quando um status muda, um evento é gravado atomicamente na tabela `webhook_outbox` dentro da mesma transação SQL. Um worker separado lê essa tabela em polling de 2 segundos e dispara chamadas HTTP para os endpoints cadastrados pelos clientes. O payload é assinado com HMAC-SHA256, e entregas com falha são reprocessadas com backoff exponencial antes de ir para uma Dead Letter Queue gerenciada via endpoint admin.

---

### Contexto e problema

Público-alvo
- Clientes B2B integrados via API (Atlas Comercial, MaxDistribuição, Nova Cargo)
- Equipe interna de operações (acesso ao endpoint admin de replay de DLQ)

Cenários de uso chave
- Cliente B2B quer saber em até 10 segundos que um pedido mudou de status, sem precisar chamar `GET /orders` repetidamente
- Operador interno precisa reprocessar manualmente um evento que ficou na Dead Letter Queue após esgotar as tentativas de entrega

Onde essa feature será implantada
- Sistema existente: monólito Node.js com banco MySQL. A feature entra como novo módulo `src/modules/webhooks/` e novo processo `src/worker.ts`, reaproveitando a infraestrutura existente (Prisma, Pino, AppError, error middleware, padrão de módulos e schemas Zod)

Problemas priorizados
- Clientes B2B fazem polling em `GET /orders` repetidamente para detectar mudanças de status, tornando a integração lenta e cara para eles (impacto: alto — a Atlas Comercial ameaçou migrar para concorrente caso a solução não seja entregue até o fim do trimestre)
- Ausência de mecanismo de notificação outbound força os clientes a manter lógica manual de detecção de mudança (impacto: médio — aumenta custo de integração e fragilidade operacional dos clientes)

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar clientes B2B sobre mudança de status dos pedidos sem polling | Latência entre mudança de status e disparo do webhook | Abaixo de 10 segundos em condições normais |
| Garantir que nenhuma mudança de status seja perdida sem evento correspondente | Consistência entre mudanças de status e eventos na outbox | Zero inconsistências (garantia atômica via transação SQL) |

---

### Escopo

Incluso
- CRUD de configuração de endpoints de webhook (cadastro, edição, remoção, listagem)
- Filtro de eventos por status na inserção na outbox
- Worker de disparo com polling de 2 segundos
- Retry com backoff exponencial: 5 tentativas (1min, 5min, 30min, 2h, 12h)
- Dead Letter Queue em tabela `webhook_dead_letter`
- Endpoint admin para replay manual de eventos da DLQ (role ADMIN)
- Assinatura HMAC-SHA256 com secret por endpoint
- Rotação de secret com grace period de 24 horas
- Histórico de entregas por endpoint (`GET /webhooks/:id/deliveries`)
- Auditoria de replay (log de quem acionou)
- Idempotência via `X-Event-Id` (UUID por evento)

Fora de escopo
- Alertas por email quando webhook falha repetidamente (adiado para próxima fase)
- Rate limiting de envio de webhooks (observar e implementar se virar problema)
- Dashboard visual para clientes (projeto separado do time de frontend)
- Garantia de ordenação global de eventos entre múltiplos workers
- Webhooks inbound (clientes enviando eventos para o sistema)

---

### Requisitos funcionais

#### FR-001 Cadastro de endpoint de webhook
O sistema deve permitir que um cliente autenticado cadastre um endpoint de webhook informando a URL de destino, o `customer_id` e a lista de status que deseja receber. A secret é gerada pelo sistema e devolvida na resposta de criação.

**Fluxo principal**
- Cliente envia `POST /webhooks` com `url`, `customer_id` e lista de status desejados
- Sistema valida que a URL usa HTTPS
- Sistema gera UUID para o webhook e gera a secret
- Sistema persiste a configuração no banco
- Sistema retorna os dados do webhook criado incluindo a secret

**Fluxos alternativos e exceções**
- Se o cliente não informar nenhum status na lista de filtro, o comportamento deve ser definido antes da implementação (hipótese: recebe todos os eventos)

**Erros previstos**
- URL com esquema HTTP recusada com erro de validação (`WEBHOOK_INVALID_URL`)
- `customer_id` ausente resulta em erro de validação (`WEBHOOK_SECRET_REQUIRED`)

**Prioridade:** alta

---

#### FR-002 Edição e remoção de endpoint de webhook
O sistema deve permitir editar a configuração de um webhook existente e removê-lo.

**Fluxo principal**
- Cliente envia `PATCH /webhooks/:id` com os campos a alterar
- Sistema valida e persiste as alterações
- Para remoção: cliente envia `DELETE /webhooks/:id` e o sistema remove o registro

**Fluxos alternativos e exceções**
- Tentativa de editar ou remover webhook de outro `customer_id` deve ser recusada

**Erros previstos**
- Webhook não encontrado retorna `WEBHOOK_NOT_FOUND`

**Prioridade:** alta

---

#### FR-003 Listagem de endpoints de webhook de um cliente
O sistema deve permitir listar todos os webhooks cadastrados de um cliente.

**Fluxo principal**
- Cliente envia `GET /webhooks?customerId=` com o `customer_id`
- Sistema retorna lista de webhooks do cliente com seus dados de configuração (sem expor a secret)

**Fluxos alternativos e exceções**
- Nenhum webhook cadastrado retorna lista vazia

**Erros previstos**
- `customer_id` ausente resulta em erro de validação

**Prioridade:** média

---

#### FR-004 Publicação atômica de evento na outbox
Quando o status de um pedido muda, o sistema deve inserir um evento na tabela `webhook_outbox` dentro da mesma transação SQL que atualiza o pedido, o histórico de status e o estoque.

**Fluxo principal**
- Método `changeStatus` em `src/modules/orders/order.service.ts` inicia transação SQL
- Dentro da transação: atualiza `orders`, insere em `order_status_history`, decrementa `stock_quantity`
- Ainda dentro da mesma transação: chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` que verifica se algum webhook do cliente está inscrito naquele status e insere o evento na `webhook_outbox` com o payload já renderizado (snapshot do estado no momento da inserção)
- Transação commitada
- Se nenhum webhook do cliente está inscrito naquele status, nada é inserido na outbox

**Fluxos alternativos e exceções**
- Se a transação principal der rollback, o registro na outbox some junto, sem inconsistência

**Erros previstos**
- Falha na inserção da outbox causa rollback da transação inteira (mudança de status não ocorre)

**Prioridade:** alta

---

#### FR-005 Worker de disparo de webhooks
Um processo Node.js separado (`src/worker.ts`) deve rodar em loop de 2 segundos, lendo eventos pendentes da `webhook_outbox` e disparando chamadas HTTP para os endpoints dos clientes.

**Fluxo principal**
- Worker busca batch de eventos com status `pendente` ordenados por `created_at`
- Para cada evento: monta o payload (já armazenado como snapshot), assina com HMAC-SHA256 usando a secret do endpoint, envia `POST` para a URL do cliente com os headers obrigatórios
- Se o cliente responde com sucesso (2xx): marca evento como `entregue` na outbox e registra no histórico de entregas
- Se o cliente responde com erro ou timeout (10 segundos): aplica lógica de retry

**Fluxos alternativos e exceções**
- Worker processa em ordem de `created_at`, garantindo ordenação implícita por `order_id` enquanto houver single-worker

**Erros previstos**
- Timeout de 10 segundos sem resposta do cliente é tratado como falha e aciona retry

**Prioridade:** alta

---

#### FR-006 Retry com backoff exponencial e Dead Letter Queue
Eventos não entregues devem ser reprocessados com backoff exponencial. Após 5 tentativas sem sucesso, o evento é movido para a `webhook_dead_letter`.

**Fluxo principal**
- Após falha: incrementa contador de tentativas e agenda próximo retry conforme progressão 1min, 5min, 30min, 2h, 12h
- Após 5ª tentativa sem sucesso: move o evento para a tabela `webhook_dead_letter` com payload, motivo da falha e timestamp
- Evento na outbox principal é marcado como falhado definitivamente

**Fluxos alternativos e exceções**
- Evento na DLQ pode ser recolocado na outbox como pendente via endpoint admin

**Erros previstos**
- Falha em todas as 5 tentativas resulta em movimentação para DLQ sem exception propagada para o worker

**Prioridade:** alta

---

#### FR-007 Replay manual de eventos da Dead Letter Queue
Um endpoint admin deve permitir reprocessar manualmente um evento da DLQ, recolocando-o na outbox como pendente.

**Fluxo principal**
- Usuário autenticado com role ADMIN envia `POST /admin/webhooks/dead-letter/:id/replay`
- Sistema valida a role ADMIN via `requireRole` existente
- Sistema move o evento da `webhook_dead_letter` de volta para `webhook_outbox` com status `pendente`
- Sistema registra em log de auditoria quem acionou o replay, com timestamp

**Fluxos alternativos e exceções**
- Somente usuários ADMIN chegam a esse endpoint

**Erros previstos**
- Evento não encontrado na DLQ retorna erro
- Usuário sem role ADMIN retorna erro de autorização

**Prioridade:** alta

---

#### FR-008 Rotação de secret do webhook
O sistema deve permitir que o cliente solicite a rotação da secret de um endpoint de webhook. A secret antiga permanece válida por 24 horas após a rotação.

**Fluxo principal**
- Cliente solicita rotação via endpoint de rotação
- Sistema gera nova secret e armazena ambas (nova e antiga com timestamp de expiração)
- Durante as 24 horas de grace period, o sistema aceita assinatura com qualquer uma das duas secrets
- Após 24 horas, a secret antiga é invalidada

**Fluxos alternativos e exceções**
- Cliente deve atualizar seus sistemas para usar a nova secret durante o grace period

**Erros previstos**
- Tentativa de assinar com secret expirada resulta em falha de validação do lado do cliente

**Prioridade:** alta

---

#### FR-009 Histórico de entregas por endpoint
O sistema deve expor o histórico de entregas de um webhook específico, incluindo sucesso ou falha, payload enviado, resposta recebida e tempo de resposta.

**Fluxo principal**
- Cliente envia `GET /webhooks/:id/deliveries`
- Sistema retorna lista de entregas com status, payload, resposta do cliente e tempo de resposta

**Fluxos alternativos e exceções**
- Nenhuma entrega registrada retorna lista vazia

**Erros previstos**
- Webhook não encontrado retorna `WEBHOOK_NOT_FOUND`

**Prioridade:** média

---

### Requisitos não funcionais

Performance
- Latência entre mudança de status e disparo do webhook inferior a 10 segundos em condições normais (determinada principalmente pelo intervalo de polling do worker de 2 segundos)
- Timeout de 10 segundos por chamada HTTP para o endpoint do cliente

Disponibilidade
- O worker deve rodar como processo separado da API, de modo que reinicializações da API não interrompam o processamento da outbox

Segurança e autorização
- Todos os endpoints de configuração de webhook exigem autenticação
- Endpoint de replay da DLQ exige role ADMIN, verificado via `requireRole` existente
- URLs de webhook devem obrigatoriamente usar HTTPS; URLs com HTTP são recusadas em validação Zod
- Cada endpoint de webhook possui sua própria secret; não existe secret global
- Payload assinado com HMAC-SHA256 usando a secret do endpoint
- Limite de 64KB por payload; payloads acima desse limite resultam em erro
- Replay attack detectável pelo cliente via header `X-Timestamp`
- Auditoria de quem acionou o replay registrada em log

Observabilidade
- Logger Pino já existente reaproveitado sem alterações
- Erros do módulo de webhooks seguem prefixo `WEBHOOK_` nos códigos (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`)
- Error middleware centralizado existente trata AppError, Zod e Prisma sem modificação

Confiabilidade e integridade de dados
- Inserção na `webhook_outbox` deve ocorrer dentro da mesma transação SQL do `changeStatus`; falha na inserção causa rollback da mudança de status
- Payload armazenado como snapshot no momento da inserção na outbox; alterações posteriores no pedido não afetam o evento
- Garantia de entrega at-least-once; cliente deve deduplificar pelo `X-Event-Id`
- IDs da outbox em UUID, seguindo padrão do restante do projeto

Compatibilidade e portabilidade
- Módulo segue padrão de estrutura existente: `src/modules/webhooks/` com controller, service, repository, routes e schemas Zod
- Worker como entry-point separada `src/worker.ts` com script `npm run worker`
- Worker usa instância própria do PrismaClient (mesmo banco, mesmo `DATABASE_URL`, processo separado)

---

### Arquitetura e abordagem

Abordagem
- Padrão Outbox sobre MySQL existente: evento gravado atomicamente na mesma transação SQL do `changeStatus`. Worker Node.js separado em polling de 2 segundos realiza os disparos HTTP. Sem introdução de nova infraestrutura (sem Redis, sem message broker externo).

Componentes
- `src/modules/webhooks/`: módulo com controller, service, repository, routes e schemas (CRUD de configuração, histórico de entregas, rotação de secret)
- `src/worker.ts`: entry-point do processo worker com script `npm run worker`
- `src/modules/webhooks/webhook.processor.ts` (ou equivalente): lógica de processamento da outbox, retry e movimentação para DLQ
- Tabela `webhook_outbox`: registros de eventos pendentes, processando, entregues e falhados, com índice em status e `created_at`
- Tabela `webhook_dead_letter`: eventos que esgotaram todas as tentativas, com payload, motivo e timestamp
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)`: função pura que recebe o client de transação ativo e insere na outbox dentro da transação do `changeStatus`

Integrações
- `src/modules/orders/order.service.ts`: método `changeStatus` chama `publishWebhookEvent` dentro da transação existente
- Endpoints HTTP externos dos clientes B2B: destino das chamadas de disparo do worker

---

### Decisões e trade-offs

#### Decisão: Padrão Outbox em MySQL em vez de disparo síncrono ou Redis Streams
- **Justificativa:** Disparo síncrono travaria a transação de mudança de status se o cliente estivesse lento ou offline e tornaria impossível o rollback. Redis Streams exigiria nova infraestrutura para um time pequeno. O MySQL já existe e o padrão Outbox resolve o problema sem dependências externas.
- **Trade-off:** Latência mínima de 2 segundos determinada pelo polling do worker. Ausência de push reativo (MySQL não tem LISTEN/NOTIFY como Postgres).

#### Decisão: Worker separado em polling de 2 segundos
- **Justificativa:** MySQL não possui mecanismo nativo de notificação para processos externos. Polling de 2 segundos atende o requisito de entrega abaixo de 10 segundos. Worker em processo separado evita que reinicializações da API interrompam o processamento.
- **Trade-off:** Consumo contínuo de conexão ao banco mesmo em períodos sem eventos. Ordenação global de eventos não garantida caso o worker seja escalado horizontalmente no futuro.

#### Decisão: DLQ em tabela separada `webhook_dead_letter` em vez de marcação na outbox principal
- **Justificativa:** Tabela separada mantém a outbox principal limpa e facilita debug e reprocessamento. Evidência de falha fica isolada sem poluir o fluxo principal de processamento.
- **Trade-off:** Requer join ou consulta adicional para visão consolidada de todos os eventos de um cliente.

#### Decisão: HMAC-SHA256 com secret por endpoint
- **Justificativa:** Secret global comprometeria todos os clientes em caso de vazamento (precedente real citado na reunião). HMAC-SHA256 é padrão de mercado com ampla disponibilidade de bibliotecas.
- **Trade-off:** Gerenciamento de N secrets (uma por endpoint cadastrado) em vez de uma global.

#### Decisão: Garantia at-least-once com idempotência via X-Event-Id
- **Justificativa:** Exactly-once exigiria coordenação dos dois lados e aumentaria significativamente a complexidade. At-least-once com `X-Event-Id` é o padrão adotado por Stripe e GitHub e resolve 99% dos casos.
- **Trade-off:** Responsabilidade de deduplicação transferida para o cliente. Cliente pode receber o mesmo evento mais de uma vez em caso de falha do worker após disparo mas antes de confirmação.

#### Decisão: Snapshot do payload na inserção na outbox
- **Justificativa:** Garante que o evento reflete o estado do pedido no momento da mudança de status, mesmo que o pedido sofra alterações posteriores.
- **Trade-off:** Espaço em disco maior por armazenar payloads completos em vez de apenas referências.

#### Decisão: 5 tentativas com backoff 1min/5min/30min/2h/12h
- **Justificativa:** 3 tentativas cobriria apenas ~30 minutos, insuficiente para manutenções planejadas de clientes. A progressão de ~15 horas cobre indisponibilidades prolongadas sem manter eventos pendentes indefinidamente.
- **Trade-off:** Eventos podem permanecer em processamento por até ~15 horas antes de ir para DLQ.

---

### Dependências

#### Técnica: Integração com `order.service.ts`
A função `publishWebhookEvent(tx, order, fromStatus, toStatus)` precisa ser chamada dentro do método `changeStatus` de `src/modules/orders/order.service.ts`. Essa alteração é o ponto de integração mais crítico: sem ela, a garantia de consistência atômica entre mudança de status e registro do evento é perdida.

#### Técnica: Modelagem das tabelas `webhook_outbox` e `webhook_dead_letter`
As tabelas precisam existir no banco antes do início do desenvolvimento do worker e do módulo. Índices em `status` e `created_at` na outbox são necessários para performance do polling.

#### Organizacional: Revisão de segurança por Sofia antes do deploy
Sofia (Eng. de Segurança) deve revisar o código de geração de secret e implementação do HMAC antes de qualquer deploy em produção. Reservar ao menos dois dias úteis para essa revisão ao final do desenvolvimento.

---

### Riscos e mitigação

#### Falha na inserção da outbox dentro da transação causa bloqueio de mudança de status
- **Probabilidade:** baixa
- **Impacto:** Mudança de status do pedido não ocorre (rollback), impactando operação do cliente B2B diretamente
- **Mitigação:**
  - Testar extensivamente o caminho de inserção na outbox em conjunto com a transação de mudança de status
  - Garantir que erros da outbox sejam capturados e logados com contexto suficiente para diagnóstico rápido
- **Plano de contingência:** Em caso de bug crítico, reverter a chamada a `publishWebhookEvent` da transação temporariamente e processar eventos retroativos manualmente via replay da DLQ após correção

#### Worker processa eventos fora de ordem em eventual escala horizontal
- **Probabilidade:** baixa (single-worker por ora)
- **Impacto:** Cliente recebe eventos de um mesmo pedido fora da sequência cronológica
- **Mitigação:**
  - Documentar como limitação conhecida aceita enquanto houver single-worker
  - Incluir `timestamp` no payload para que o cliente possa ordenar do lado dele se necessário
- **Plano de contingência:** Se escala horizontal for necessária, implementar particionamento por `order_id` ou lock pessimista antes de colocar segundo worker em produção

#### Vazamento de secret de webhook pelo cliente
- **Probabilidade:** média (precedente real citado na reunião)
- **Impacto:** Terceiro pode forjar eventos para o endpoint do cliente comprometido
- **Mitigação:**
  - Secret por endpoint (não global) limita o raio do vazamento a um único endpoint
  - Endpoint de rotação de secret com grace period de 24h disponível para resposta rápida
- **Plano de contingência:** Cliente aciona rotação imediata da secret; sistema invalida a secret antiga após 24h mesmo que o cliente não complete a migração

#### Acúmulo de eventos na outbox em pico de mudanças de status
- **Probabilidade:** média
- **Impacto:** Aumento de latência de entrega acima do requisito de 10 segundos em momentos de pico
- **Mitigação:**
  - Monitorar tamanho da outbox e latência média de processamento desde o início
  - Garantir índices adequados na tabela antes do go-live
- **Plano de contingência:** Aumentar tamanho do batch do worker ou reduzir intervalo de polling como ajuste operacional emergencial

---

### Critérios de aceitação

- `POST /webhooks` cria o endpoint, gera a secret e retorna os dados completos incluindo a secret gerada
- URL com esquema HTTP é recusada com erro de validação em `POST /webhooks` e `PATCH /webhooks/:id`
- Mudança de status de pedido que possui webhook inscrito naquele status resulta em exatamente um registro na `webhook_outbox` com payload snapshot correto
- Mudança de status de pedido sem nenhum webhook inscrito naquele status não gera nenhum registro na `webhook_outbox`
- Se a inserção na `webhook_outbox` falhar, a mudança de status do pedido não ocorre (rollback confirmado por teste)
- Worker dispara chamada HTTP para o endpoint do cliente em até 10 segundos após a mudança de status em condições normais
- Chamada HTTP enviada pelo worker inclui os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`
- Assinatura `X-Signature` é HMAC-SHA256 do corpo da requisição usando a secret do endpoint e pode ser verificada pelo cliente
- Payload enviado ao cliente é limitado a 64KB; payloads acima disso resultam em erro
- Falha na entrega aciona retry com a progressão 1min, 5min, 30min, 2h, 12h (5 tentativas total)
- Após 5 tentativas sem sucesso, o evento aparece na tabela `webhook_dead_letter` com payload, motivo da falha e timestamp
- `POST /admin/webhooks/dead-letter/:id/replay` é acessível somente com role ADMIN; retorna erro de autorização para qualquer outra role
- Replay de evento da DLQ gera registro de auditoria com identificação de quem acionou e timestamp
- Durante grace period de rotação de secret, assinatura com a secret antiga e com a nova são ambas aceitas pelo sistema
- Após 24 horas do grace period, assinatura com a secret antiga é invalidada
- `GET /webhooks/:id/deliveries` retorna histórico com status de entrega, payload enviado, resposta do cliente e tempo de resposta
- Dois eventos com o mesmo conteúdo recebem `X-Event-Id` distintos

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para a função `publishWebhookEvent`, lógica de filtro de status na inserção da outbox, geração e verificação de HMAC-SHA256, e lógica de backoff exponencial
- Testes de integração para o fluxo completo: mudança de status do pedido → inserção na outbox → disparo pelo worker → registro no histórico de entregas
- Teste de integridade transacional: falha intencional na inserção da outbox deve causar rollback da mudança de status
- Teste de segurança: verificar que endpoint de replay da DLQ rejeita roles diferentes de ADMIN, que URLs HTTP são recusadas, e que assinatura inválida pode ser detectada pelo cliente
- Teste de retry: simular cliente offline para validar a progressão de backoff e movimentação para DLQ após 5 tentativas

Estratégia de validação
- TDD para lógica crítica de transação (inserção atômica na outbox) e HMAC
- QA manual guiado por roteiro cobrindo os fluxos de cadastro, disparo, retry, DLQ e replay
- Revisão de segurança por Sofia (Eng. de Segurança) antes do deploy em produção, com foco em geração de secret e implementação do HMAC
### PRD: Order Management System — Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: novembro/2024
Responsável: Larissa (Tech Lead) / Marcos (PM)

---

### Resumo

O OMS não possui mecanismo de notificação proativa para sistemas externos. Clientes B2B que precisam reagir a mudanças de status de pedidos são obrigados a fazer polling contínuo na API, gerando tráfego desnecessário e latência percebida alta.

Esta feature introduz um sistema de webhooks outbound: quando o status de um pedido muda, o sistema notifica automaticamente os endpoints cadastrados pelos clientes via HTTP POST, com payload assinado e garantia de entrega com retry automático.

---

### Contexto e problema

Público-alvo
- Clientes B2B integrados via API que precisam reagir automaticamente a mudanças de status de pedidos
- Equipe interna de operações que precisa intervir manualmente em falhas de entrega

Cenários de uso chave
- Cliente B2B recebe notificação automática quando um pedido muda de status e atualiza seu ERP sem intervenção manual
- Cliente recebe evento de envio (`SHIPPED`) e dispara notificação de rastreamento para seu consumidor final
- Operador interno reprocessa manualmente um evento que ficou na fila de falhas após indisponibilidade do cliente

Onde essa feature será implantada
- Sistema existente: Order Management System (OMS) em produção. A feature entra como novo módulo de webhooks e novo processo worker, sem introdução de nova infraestrutura externa.

Problemas priorizados
- Clientes B2B fazem polling repetido no endpoint de listagem de pedidos para detectar mudanças de status, tornando a integração lenta e cara (impacto: alto — Atlas Comercial ameaçou migrar para concorrente caso a solução não seja entregue até o fim do trimestre)
- Ausência de notificação proativa força clientes a manter lógica manual de detecção de mudança, aumentando custo de integração e fragilidade operacional (impacto: médio)

---

### Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Notificar clientes B2B sobre mudança de status sem polling | Latência entre mudança de status e disparo do webhook | Abaixo de 10 segundos em condições normais |
| Reter clientes B2B que solicitaram a feature | Clientes integrados via webhook após o lançamento | 3 de 3 (Atlas Comercial, MaxDistribuição, Nova Cargo) |
| Reduzir tráfego de polling na API | Volume de chamadas ao endpoint de listagem de pedidos pelos clientes integrados | Redução mensurável em 30 dias após o lançamento |
| Garantir confiabilidade de entrega | Taxa de entrega bem-sucedida na primeira tentativa | Acima de 95% em condições normais de operação |

---

### Escopo

Incluso
- Cadastro, edição, remoção e listagem de endpoints de webhook por cliente
- Entrega de notificações via HTTP POST quando o status de um pedido muda
- Filtro por status: cliente escolhe quais transições de status deseja receber
- Assinatura do payload com chave secreta por endpoint (HMAC-SHA256)
- Retry automático com backoff exponencial em caso de falha de entrega
- Fila de eventos não entregues (Dead Letter Queue) com tabela dedicada
- Replay manual de eventos da DLQ por operadores autorizados
- Histórico de entregas por endpoint (status, payload, resposta, tempo)
- Rotação de chave secreta com período de transição de 24 horas

Fora de escopo
- Notificação por e-mail ao cliente quando webhooks falham repetidamente — adiado para fase futura após medir impacto da feature principal `[09:37] Larissa` `[09:38] Marcos`
- Rate limiting de envio de webhooks por cliente — observar em produção e implementar se necessário `[09:39] Diego`
- Dashboard visual para clientes acompanharem webhooks — projeto separado do time de frontend `[09:40] Larissa`
- Webhooks inbound (cliente enviando eventos para o sistema) — fora de escopo; feature é exclusivamente outbound `[09:02] Sofia`
- Garantia de ordenação global de eventos entre diferentes pedidos — limitação conhecida e aceita `[09:13] Larissa`

---

### Requisitos funcionais

#### FR-001 Cadastro de endpoint de webhook
O sistema deve permitir que um cliente autenticado cadastre um endpoint informando a URL de destino, o identificador do cliente e a lista de status que deseja receber. A chave secreta é gerada pelo sistema e entregue apenas no momento da criação.

**Fluxo principal**
- Cliente envia requisição de criação com URL, identificador do cliente e lista de status de interesse
- Sistema valida que a URL usa protocolo seguro (HTTPS)
- Sistema gera identificador único e chave secreta para o endpoint
- Sistema persiste a configuração
- Sistema retorna os dados do endpoint criado, incluindo a chave secreta gerada

**Fluxos alternativos e exceções**
- Se nenhum status for informado na lista de filtro, o comportamento deve ser definido antes da implementação (hipótese: recebe todos os eventos)

**Erros previstos**
- URL com protocolo HTTP é recusada com erro de validação
- Identificador do cliente ausente resulta em erro de validação

**Prioridade:** alta

---

#### FR-002 Edição de endpoint de webhook
O sistema deve permitir que o cliente edite a configuração de um endpoint existente, como URL e filtro de status.

**Fluxo principal**
- Cliente envia requisição de edição com os campos a alterar
- Sistema valida os dados e persiste as alterações

**Fluxos alternativos e exceções**
- Tentativa de editar endpoint de outro cliente deve ser recusada

**Erros previstos**
- Endpoint não encontrado retorna erro de recurso não encontrado
- URL editada para protocolo HTTP é recusada com erro de validação

**Prioridade:** alta

---

#### FR-003 Remoção de endpoint de webhook
O sistema deve permitir que o cliente remova um endpoint cadastrado.

**Fluxo principal**
- Cliente envia requisição de remoção identificando o endpoint
- Sistema remove o registro e cessa futuros disparos para aquela URL

**Fluxos alternativos e exceções**
- Tentativa de remover endpoint de outro cliente deve ser recusada

**Erros previstos**
- Endpoint não encontrado retorna erro de recurso não encontrado

**Prioridade:** alta

---

#### FR-004 Listagem de endpoints de webhook de um cliente
O sistema deve permitir listar todos os endpoints de webhook cadastrados para um cliente.

**Fluxo principal**
- Cliente envia requisição de listagem informando o identificador do cliente
- Sistema retorna lista de endpoints com seus dados de configuração, sem expor a chave secreta

**Fluxos alternativos e exceções**
- Cliente sem webhooks cadastrados recebe lista vazia

**Erros previstos**
- Identificador do cliente ausente resulta em erro de validação

**Prioridade:** média

---

#### FR-005 Notificação de mudança de status via webhook
Quando o status de um pedido muda, o sistema deve disparar uma notificação HTTP POST para todos os endpoints do cliente que estejam inscritos naquele status, com o payload representando o estado do pedido no momento da mudança.

**Fluxo principal**
- Status de um pedido muda
- Sistema verifica quais endpoints do cliente estão inscritos naquele status
- Para cada endpoint inscrito: sistema registra o evento e dispara chamada HTTP POST com payload assinado
- Se o cliente responde com sucesso, entrega é registrada como concluída

**Fluxos alternativos e exceções**
- Se nenhum endpoint do cliente está inscrito naquele status, nenhuma notificação é gerada
- O registro do evento deve ocorrer de forma atômica junto com a mudança de status: se a mudança não ocorrer, o evento não é gerado; se o evento não puder ser registrado, a mudança não ocorre

**Erros previstos**
- Falha na entrega ao endpoint do cliente aciona o fluxo de retry

**Prioridade:** alta

---

#### FR-006 Retry automático com backoff exponencial
Eventos não entregues devem ser reprocessados automaticamente com intervalos crescentes entre tentativas.

**Fluxo principal**
- Após falha de entrega: sistema agenda nova tentativa após 1 minuto
- Tentativas subsequentes: 5 minutos, 30 minutos, 2 horas, 12 horas
- Total de 5 tentativas antes de considerar a entrega como falha permanente
- Após esgotar todas as tentativas: evento é movido para a fila de falhas (DLQ)

**Fluxos alternativos e exceções**
- Evento que chega à DLQ pode ser reprocessado manualmente via endpoint admin

**Erros previstos**
- Resposta do cliente com código de erro HTTP é tratada como falha e aciona retry
- Ausência de resposta dentro do tempo limite é tratada como falha e aciona retry

**Prioridade:** alta

---

#### FR-007 Dead Letter Queue e replay manual
Eventos que esgotaram todas as tentativas de entrega devem ser persistidos em fila de falhas separada. Um operador autorizado deve conseguir reprocessar manualmente um evento dessa fila.

**Fluxo principal**
- Evento com 5 falhas é movido para a DLQ com registro de payload, motivo e timestamp
- Operador com permissão de administrador acessa endpoint de replay
- Sistema recoloca o evento na fila principal como pendente
- Sistema registra em auditoria quem acionou o replay e quando

**Fluxos alternativos e exceções**
- Somente usuários com perfil de administrador podem acionar o replay

**Erros previstos**
- Evento não encontrado na DLQ retorna erro
- Usuário sem permissão de administrador recebe erro de autorização

**Prioridade:** alta

---

#### FR-008 Histórico de entregas por endpoint
O sistema deve expor o histórico de entregas de cada endpoint, permitindo que o cliente audite o que foi enviado, quando e com qual resultado.

**Fluxo principal**
- Cliente solicita histórico informando o identificador do endpoint
- Sistema retorna lista de tentativas de entrega com status, payload enviado, resposta recebida e tempo de resposta

**Fluxos alternativos e exceções**
- Nenhuma entrega registrada retorna lista vazia

**Erros previstos**
- Endpoint não encontrado retorna erro de recurso não encontrado

**Prioridade:** média

---

#### FR-009 Rotação de chave secreta
O cliente deve conseguir solicitar a troca da chave secreta de um endpoint. A chave anterior permanece válida por 24 horas para permitir que o cliente migre seus sistemas sem interrupção.

**Fluxo principal**
- Cliente solicita rotação de chave para um endpoint
- Sistema gera nova chave e armazena junto com a anterior e seu prazo de validade
- Durante 24 horas: sistema aceita assinatura feita com qualquer uma das duas chaves
- Após 24 horas: chave anterior é invalidada

**Fluxos alternativos e exceções**
- Cliente deve atualizar seus sistemas para usar a nova chave durante o período de transição

**Erros previstos**
- Tentativa de validar com chave expirada resulta em falha de verificação do lado do cliente

**Prioridade:** alta

---

### Requisitos não funcionais

Performance
- Latência entre mudança de status e disparo da notificação inferior a 10 segundos em condições normais de operação
- Tempo máximo de espera por resposta do endpoint do cliente de 10 segundos por tentativa de entrega

Disponibilidade
- O processo de entrega de webhooks deve ser independente do processo da API principal, de modo que reinicializações da API não interrompam notificações em andamento

Segurança e autorização
- Todos os endpoints de configuração de webhook exigem autenticação
- Replay de eventos da DLQ exige perfil de administrador
- URLs de webhook devem obrigatoriamente usar HTTPS; URLs com HTTP são recusadas na validação
- Cada endpoint possui chave secreta própria; não existe chave global compartilhada entre clientes
- Payload de cada notificação é assinado com a chave secreta do endpoint, permitindo que o cliente verifique autenticidade
- Tamanho máximo de payload por notificação: 64KB; payloads maiores resultam em erro, não em truncamento
- Cada notificação inclui timestamp de envio, permitindo que o cliente detecte ataques de replay se necessário

Observabilidade
- Histórico de entregas disponível por endpoint com status, payload, resposta e tempo de resposta
- Auditoria de replay com identificação do operador e timestamp

Confiabilidade e integridade de dados
- O registro do evento de notificação deve ser atômico com a mudança de status do pedido: as duas operações ocorrem ou nenhuma ocorre
- O payload armazenado representa o estado do pedido no momento da mudança, não no momento do envio
- Garantia de entrega at-least-once: cliente pode receber o mesmo evento mais de uma vez; cada evento carrega identificador único para deduplicação do lado do cliente

Compatibilidade e portabilidade
- Notificações entregues via HTTP POST com payload JSON
- Identificadores de eventos em formato UUID, seguindo padrão existente do sistema

---

### Arquitetura e abordagem

Abordagem
- Padrão Outbox: evento de notificação registrado atomicamente no banco de dados junto com a mudança de status, eliminando risco de inconsistência. Processo worker separado lê os eventos pendentes e realiza os disparos HTTP. Sem introdução de infraestrutura externa (sem fila de mensagens, sem cache dedicado).

Componentes
- Módulo de configuração de webhooks: CRUD de endpoints, rotação de secret, histórico de entregas
- Fila de saída (outbox): tabela de eventos pendentes com suporte a retry e controle de status
- Processo worker: responsável por ler a fila, assinar e disparar as notificações, registrar resultados e gerenciar retries
- Fila de falhas (DLQ): tabela dedicada para eventos que esgotaram todas as tentativas, com suporte a replay manual

Integrações
- Ciclo de mudança de status do pedido: ponto onde o evento é gerado e registrado atomicamente
- Endpoints HTTP dos clientes B2B: destino das notificações disparadas pelo worker

---

### Decisões e trade-offs

#### Decisão: Padrão Outbox em banco de dados existente em vez de disparo síncrono ou fila externa
- **Justificativa:** Disparo síncrono travaria a operação de mudança de status se o cliente estivesse lento ou offline, e tornaria impossível o rollback. Fila de mensagens externa exigiria nova infraestrutura para um time pequeno. O banco existente resolve o problema sem dependências adicionais.
- **Trade-off:** Latência mínima de entrega determinada pelo intervalo de polling do worker. Banco de dados assume papel de fila, o que pode gerar pressão em volumes muito altos.

#### Decisão: Worker em processo separado com polling a cada 2 segundos
- **Justificativa:** O banco de dados existente não possui mecanismo nativo de notificação para processos externos. Polling a cada 2 segundos atende o requisito de entrega abaixo de 10 segundos. Processo separado garante que reinicializações da API não interrompam entregas em andamento.
- **Trade-off:** Consumo contínuo de conexão ao banco mesmo em períodos sem eventos. Ordenação global de eventos entre pedidos diferentes não é garantida caso o worker seja escalado horizontalmente no futuro.

#### Decisão: Fila de falhas em tabela separada em vez de marcação na fila principal
- **Justificativa:** Tabela dedicada mantém a fila principal limpa e facilita debug, reprocessamento e rastreabilidade de falhas sem poluir o fluxo de processamento normal.
- **Trade-off:** Requer consulta adicional para visão consolidada de todos os eventos de um cliente.

#### Decisão: Chave secreta por endpoint em vez de chave global
- **Justificativa:** Chave global comprometida afetaria todos os clientes simultaneamente. Um precedente real de vazamento de chave em log de cliente foi citado como motivação. Chave por endpoint limita o raio de impacto de um eventual vazamento.
- **Trade-off:** Gerenciamento de múltiplas chaves (uma por endpoint cadastrado) em vez de uma única chave global.

#### Decisão: Garantia at-least-once com identificador único por evento
- **Justificativa:** Garantia exactly-once exigiria coordenação entre os dois lados e aumentaria significativamente a complexidade. At-least-once com identificador único por evento é o padrão adotado por Stripe e GitHub e resolve a grande maioria dos casos práticos.
- **Trade-off:** Responsabilidade de deduplicação transferida para o cliente. Cliente pode receber o mesmo evento mais de uma vez em cenários de falha do worker após o disparo mas antes do registro de confirmação.

#### Decisão: 5 tentativas com progressão 1min / 5min / 30min / 2h / 12h
- **Justificativa:** 3 tentativas cobriria apenas cerca de 30 minutos, insuficiente para janelas de manutenção planejada de clientes. A progressão de aproximadamente 15 horas cobre indisponibilidades prolongadas sem manter eventos pendentes indefinidamente.
- **Trade-off:** Eventos podem permanecer em processamento por até 15 horas antes de ir para a fila de falhas.

#### Decisão: Payload armazenado como snapshot no momento do registro
- **Justificativa:** Garante que a notificação representa o estado do pedido no momento em que a mudança ocorreu, mesmo que o pedido sofra alterações posteriores.
- **Trade-off:** Maior consumo de espaço em banco por armazenar payloads completos em vez de referências.

---

### Dependências

#### Técnica: Integração com o ciclo de mudança de status do pedido
O evento de notificação precisa ser gerado dentro da mesma operação transacional que muda o status do pedido. Essa é a dependência técnica mais crítica: sem ela, a garantia de consistência entre mudança de status e geração do evento é perdida.

#### Técnica: Modelagem do banco de dados para outbox e DLQ
As tabelas de outbox e DLQ precisam estar criadas e com índices adequados antes do início do desenvolvimento do worker. A ausência de índices adequados afeta diretamente a performance do polling.

#### Organizacional: Revisão de segurança antes do deploy
A engenheira de segurança (Sofia) deve revisar a implementação de geração de chave secreta e assinatura HMAC antes de qualquer deploy em produção. Reservar ao menos dois dias úteis para essa revisão ao final do desenvolvimento.

---

### Riscos e mitigação

#### Cliente com indisponibilidade prolongada perde notificações críticas
- **Probabilidade:** média
- **Impacto:** Eventos não entregues após esgotar as 5 tentativas ficam na DLQ. Se o cliente não acionar replay, pode perder permanentemente notificações de mudanças importantes.
- **Mitigação:**
  - Retry com progressão de até 15 horas cobre a maioria das indisponibilidades planejadas
  - Endpoint de replay manual disponível para reprocessamento após retorno do cliente
  - Histórico de entregas permite ao cliente identificar lacunas e buscar dados complementares via API
- **Plano de contingência:** Operador interno aciona replay manual dos eventos da DLQ após identificar a indisponibilidade do cliente. Se o volume for alto, o processo pode ser repetido em lote.

#### Vazamento de chave secreta pelo cliente compromete autenticidade das notificações
- **Probabilidade:** média (precedente real citado na reunião — cliente anterior vazou chave em log de aplicação)
- **Impacto:** Terceiro pode forjar notificações para o endpoint do cliente comprometido, potencialmente causando ações indevidas no sistema do cliente.
- **Mitigação:**
  - Chave por endpoint limita o raio de impacto a um único endpoint, não a todos os clientes
  - Rotação de chave com grace period de 24 horas disponível para resposta rápida ao incidente
  - Documentar claramente no portal de desenvolvedores que chaves não devem ser logadas
- **Plano de contingência:** Cliente aciona rotação imediata da chave. Sistema invalida a chave antiga após 24 horas independentemente da migração do cliente.

#### Acúmulo de eventos na fila em pico de operações degrada a latência de entrega
- **Probabilidade:** baixa no curto prazo, média se o volume crescer
- **Impacto:** Latência de entrega pode superar o requisito de 10 segundos em momentos de pico, afetando a percepção de tempo real pelos clientes.
- **Mitigação:**
  - Monitorar tamanho da fila e latência média de processamento desde o go-live
  - Garantir índices adequados nas tabelas antes do lançamento
- **Plano de contingência:** Aumentar o tamanho do batch processado pelo worker ou reduzir o intervalo de polling como ajuste operacional de emergência.

#### Cliente recebe o mesmo evento duas vezes e processa duplicado
- **Probabilidade:** média (inerente à garantia at-least-once)
- **Impacto:** Depende do sistema do cliente. Pode causar duplicação de ação no ERP, disparo duplo de notificação ao consumidor final ou inconsistência em relatórios.
- **Mitigação:**
  - Identificador único por evento enviado em header dedicado para deduplicação do lado do cliente
  - Documentar claramente o contrato at-least-once e a responsabilidade de deduplicação no portal de desenvolvedores
- **Plano de contingência:** Suporte orienta o cliente a implementar deduplicação por identificador do evento caso o problema seja reportado após o lançamento.

---

### Critérios de aceitação

- Endpoint cadastrado com URL HTTPS recebe chamada HTTP POST quando pedido do cliente muda para status configurado no filtro
- Endpoint cadastrado com URL HTTP é recusado com erro de validação no cadastro e na edição
- Notificação não disparada para mudança de status que não consta no filtro do endpoint
- Payload entregue contém identificador do evento, tipo do evento, timestamp, identificador do pedido, número do pedido, status anterior, novo status, identificador do cliente e valor total
- Header de assinatura presente em todas as entregas e verificável com a chave secreta do endpoint
- Header com identificador único do evento presente em todas as entregas; dois eventos distintos têm identificadores distintos
- Falha na entrega aciona retry com progressão exata de 1min, 5min, 30min, 2h, 12h
- Após 5 falhas consecutivas, evento aparece na fila de falhas com payload, motivo e timestamp; fila principal não é bloqueada
- Replay de evento da fila de falhas via endpoint admin recoloca o evento como pendente para nova tentativa
- Endpoint de replay acessível somente por usuário com perfil de administrador; qualquer outro perfil recebe erro de autorização
- Replay gera registro de auditoria com identificação de quem acionou e timestamp
- Rotação de chave mantém a chave anterior válida por exatamente 24 horas após a rotação
- Histórico de entregas retorna status, payload enviado, resposta do cliente e tempo de resposta por entrega
- Rollback na operação de mudança de status do pedido não deixa evento órfão na fila de saída
- Mudança de status sem nenhum endpoint inscrito naquele status não gera nenhum registro na fila de saída

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para lógica de filtro de status, geração e verificação de assinatura, e cálculo do backoff exponencial
- Testes de integração para o fluxo completo: mudança de status do pedido, geração do evento, disparo pelo worker, registro no histórico de entregas
- Teste de atomicidade: falha intencional no registro do evento deve causar rollback da mudança de status do pedido
- Teste de retry: simular cliente offline para validar a progressão de backoff e movimentação para a fila de falhas após 5 tentativas
- Teste de segurança: verificar que endpoint de replay rejeita perfis sem permissão, que URLs HTTP são recusadas, e que assinatura com chave incorreta pode ser detectada pelo cliente

Estratégia de validação
- TDD para lógica crítica de atomicidade entre mudança de status e registro do evento, e para geração e verificação de assinatura
- QA manual guiado por roteiro cobrindo os fluxos de cadastro, disparo, retry, DLQ e replay
- Revisão de segurança por Sofia (Eng. de Segurança) antes do deploy em produção, com foco em geração de chave secreta e implementação da assinatura
- Validação com Atlas Comercial, MaxDistribuição e Nova Cargo em ambiente de homologação antes do go-live

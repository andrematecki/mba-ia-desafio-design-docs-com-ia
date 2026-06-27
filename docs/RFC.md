# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | novembro/2024 |
| **Revisores** | Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Eng. Segurança), Marcos (PM) |

---

## TL;DR

Proposta de adição de sistema de webhooks outbound ao OMS para notificar clientes B2B sobre mudanças de status de pedidos. A solução usa o padrão Outbox sobre o banco MySQL existente, com worker em processo separado realizando os disparos HTTP, retry com backoff exponencial e fila de falhas. Não requer nova infraestrutura. Garante consistência atômica entre mudança de status e geração do evento.

---

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) realizam polling contínuo no endpoint de listagem de pedidos para detectar mudanças de status. Isso gera tráfego desproporcional, latência percebida alta e custo de integração elevado para os clientes. A Atlas Comercial comunicou risco de churn caso a solução não seja entregue até o fim do trimestre.

O OMS não possui nenhum mecanismo de notificação proativa. A feature precisa ser entregue sem introduzir risco na operação atual, sem dependências de nova infraestrutura e com consistência garantida entre a mudança de status do pedido e a geração da notificação.

---

## Proposta técnica

### Visão geral

A solução adota o **padrão Outbox**: ao mudar o status de um pedido, o evento de notificação é gravado atomicamente na mesma transação SQL. Um **worker em processo separado** lê essa fila em polling e realiza os disparos HTTP para os endpoints cadastrados pelos clientes.

Esse modelo desacopla completamente o disparo do webhook da operação de negócio: a transação de mudança de status não depende da disponibilidade ou latência do cliente. Se a transação der rollback, o evento some junto. Se o cliente estiver offline, o evento permanece na fila para retry.

### Componentes principais

**Fila de saída (outbox):** tabela de eventos pendentes criada atomicamente com a mudança de status. Armazena o payload já renderizado no momento da inserção — snapshot do estado do pedido naquele instante.

**Worker de disparo:** processo Node.js separado da API, em polling a cada 2 segundos. Lê eventos pendentes em batch, assina o payload com HMAC-SHA256 usando a secret do endpoint, realiza o HTTP POST e registra o resultado. Intervalo de 2 segundos atende o requisito dos clientes de notificação abaixo de 10 segundos.

**Política de retry:** backoff exponencial com 5 tentativas (1min, 5min, 30min, 2h, 12h), cobrindo uma janela de aproximadamente 15 horas. Após esgotar as tentativas, o evento é movido para fila de falhas (DLQ) em tabela separada.

**Segurança:** cada endpoint de webhook possui sua própria chave secreta, gerada pelo sistema. O payload é assinado com HMAC-SHA256. URLs de webhook devem obrigatoriamente usar HTTPS. Rotação de chave disponível com grace period de 24 horas.

**Idempotência:** cada evento recebe um identificador único (UUID) enviado ao cliente via header. O sistema garante at-least-once — o cliente usa o identificador para deduplicar em caso de entrega duplicada.

### Fluxo resumido

```
Mudança de status do pedido
        │
        ▼
[Transação SQL única]
  ├── Atualiza status do pedido
  ├── Registra histórico
  ├── Ajusta estoque
  └── Insere evento na outbox (snapshot do payload)
        │
        ▼
[Worker — polling 2s]
  ├── Lê eventos pendentes
  ├── Assina payload (HMAC-SHA256)
  ├── POST para endpoint do cliente
  ├── Sucesso → marca como entregue
  └── Falha → agenda retry (backoff) → DLQ após 5 tentativas
```

---

## Alternativas consideradas

### Alternativa 1: Disparo síncrono dentro da transação de mudança de status

**O que seria:** ao mudar o status do pedido, o sistema realizaria imediatamente a chamada HTTP ao endpoint do cliente, ainda dentro da transação.

**Por que foi descartada:** a transação de mudança de status já é pesada — atualiza o pedido, o histórico e o estoque. Adicionar uma chamada HTTP síncrona a um sistema externo tornaria o tempo de resposta dependente da latência e disponibilidade do cliente. Se o cliente estiver offline ou lento, a operação de negócio ficaria bloqueada. Rollback seria impossível sem desfazer a mudança de status. Risco direto à operação atual.

### Alternativa 2: Fila de mensagens externa (Redis Streams, RabbitMQ, Kafka)

**O que seria:** publicar eventos em uma fila de mensagens externa no momento da mudança de status, com worker consumindo dessa fila para realizar os disparos.

**Por que foi descartada:** introduz nova infraestrutura que o time precisaria operar (provisionamento, monitoramento, alta disponibilidade). Para o volume e a complexidade do problema atual, o overhead operacional não se justifica. O padrão Outbox sobre o MySQL existente resolve o mesmo problema sem dependências adicionais.

### Alternativa 3: Trigger no banco de dados para notificar o worker

**O que seria:** usar trigger no MySQL para detectar mudanças de status e notificar o worker de alguma forma, eliminando o polling.

**Por que foi descartada:** o MySQL não possui mecanismo nativo de notificação para processos externos equivalente ao LISTEN/NOTIFY do PostgreSQL. Qualquer solução sobre trigger exigiria improviso (escrita em arquivo, chamada a endpoint), adicionando complexidade sem ganho real. Polling a cada 2 segundos atende o requisito de latência e é mais simples de operar e depurar.

---

## Questões em aberto

### 1. Rate limiting de envio por cliente

Se um cliente tiver muitos pedidos mudando de status em curto período, o worker disparará chamadas em alta frequência para o mesmo endpoint. Não foi decidido se isso é problema no volume atual. Registrado como ponto a observar em produção — implementar limitação de taxa por cliente se o comportamento se mostrar problemático após o lançamento.

### 2. Escala horizontal do worker

O modelo atual assume um único processo worker. Com um único worker, a ordenação de eventos por pedido é implicitamente preservada pelo `created_at` da outbox. Se o worker for escalado horizontalmente no futuro, essa garantia é perdida — dois workers poderiam processar eventos do mesmo pedido fora de ordem.

A solução (particionamento por `order_id` ou lock pessimista) foi identificada mas deliberadamente adiada. É um problema do futuro, não do escopo atual.

### 3. Comportamento quando a lista de filtro de status está vazia

Não foi definido o que acontece se um webhook for cadastrado sem nenhum status na lista de filtro. As opções são: recusar o cadastro com erro de validação, ou interpretar lista vazia como "receber todos os eventos". Precisa ser definido antes da implementação do endpoint de cadastro.

---

## Impacto e riscos

**Impacto na operação atual:** a integração com o ciclo de mudança de status do pedido é o ponto de maior risco. O evento precisa ser inserido dentro da transação existente. Qualquer bug nessa integração pode bloquear mudanças de status em produção. Requer testes de atomicidade extensivos e revisão cuidadosa antes do deploy.

**Impacto em infraestrutura:** nenhuma nova infraestrutura. O worker usa o mesmo banco de dados via novo processo Node.js. O volume de leituras no banco aumenta com o polling do worker, mas o impacto é baixo com índices adequados na tabela de outbox.

**Risco de segurança:** o payload contém dados de pedidos enviados para endpoints externos. Chave comprometida permite forjar notificações. Mitigado pela chave por endpoint (raio de impacto limitado) e pelo mecanismo de rotação. Revisão de segurança obrigatória antes do deploy.

**Risco operacional:** cliente com indisponibilidade superior a 15 horas perde eventos para a DLQ. Endpoint de replay manual disponível para operadores com perfil de administrador.

---

## Decisões relacionadas

As decisões técnicas individuais levantadas nesta proposta estão formalizadas nos ADRs abaixo:

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling](adrs/ADR-002-worker-em-processo-separado.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — Garantia at-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-event-id.md)
- [ADR-006 — Módulo de webhooks implementado dentro do monólito existente](adrs/ADR-006-webhook-no-monolito-existente.md)
- [ADR-007 — Payload do evento armazenado como snapshot na inserção](adrs/ADR-007-snapshot-payload-na-insercao.md)

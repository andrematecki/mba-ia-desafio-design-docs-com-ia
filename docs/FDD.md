### FDD: Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: novembro/2024
Responsável: Bruno (Eng. Pleno) / Diego (Eng. Sênior)

---

### 1. Contexto e motivação técnica

O OMS não possui nenhum mecanismo de notificação proativa para sistemas externos. Clientes B2B detectam mudanças de status de pedidos via polling no endpoint `GET /orders`, gerando tráfego desnecessário e latência dependente do intervalo de pooling do cliente.

A feature adiciona ao monólito existente um sistema de webhooks outbound composto por dois elementos: um módulo de configuração e entrega (`src/modules/webhooks/`) e um processo worker separado (`src/worker.ts`). O ponto de integração crítico é o método `changeStatus` em `src/modules/orders/order.service.ts`, onde um evento precisa ser gravado atomicamente dentro da transação SQL existente.

**Atores e limites:**
- **Order Service** (`src/modules/orders/order.service.ts`): gera o evento ao mudar status do pedido
- **Módulo de webhooks** (`src/modules/webhooks/`): gerencia configurações de endpoints e histórico de entregas
- **Worker** (`src/worker.ts`): processo separado que lê a outbox e dispara as chamadas HTTP
- **Endpoints externos dos clientes B2B**: recebem as notificações via HTTP POST

**Suposições:**
- O `customer_id` é passado explicitamente no body das requisições; não é derivado do JWT (o JWT representa o operador, não o cliente)
- HTTPS é obrigatório para URLs de webhook; a validação é feita via schema Zod
- A tabela `webhook_outbox` possui índice em `(status, created_at)` para performance do polling

**Restrições:**
- Nenhuma alteração nos arquivos de código existentes além da extensão do método `changeStatus`
- O worker não pode rodar dentro do processo da API

---

### 2. Objetivos técnicos

- Inserção do evento na outbox é atômica com a mudança de status: se a transação der rollback, o evento some junto; se commitar, o evento está registrado — zero inconsistências possíveis
- Latência máxima entre mudança de status e primeira tentativa de disparo: 10 segundos (determinada pelo polling de 2 segundos do worker + tempo de processamento do batch)
- Worker opera como processo Node.js separado da API, com ciclo de vida independente
- Cada entrega carrega assinatura HMAC-SHA256 verificável pelo cliente com a secret do endpoint
- Cada evento carrega `X-Event-Id` único (UUID) para permitir deduplicação do lado do cliente (at-least-once)
- Erros do módulo de webhooks seguem o padrão `WEBHOOK_*` e são tratados pelo error middleware centralizado existente sem modificação

---

### 3. Escopo e exclusões

**Incluído**
- Tabelas `webhook_endpoint`, `webhook_outbox`, `webhook_dead_letter` e `webhook_delivery_log`
- CRUD de configuração de endpoints de webhook
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada dentro da transação do `changeStatus`
- Worker com polling de 2 segundos, batch de eventos pendentes, assinatura HMAC, HTTP POST, registro de resultado
- Retry com backoff exponencial: 5 tentativas (1min, 5min, 30min, 2h, 12h)
- Dead Letter Queue em tabela separada com endpoint de replay restrito a ADMIN
- Rotação de secret com grace period de 24 horas (duas secrets válidas simultaneamente)
- Histórico de entregas por endpoint
- Prefixo `WEBHOOK_` em todos os códigos de erro do módulo

**Excluído**
- Rate limiting de envio por cliente (observar em produção)
- Dashboard visual para clientes
- Notificação por e-mail em caso de falhas repetidas
- Garantia de ordenação global entre pedidos distintos
- Arquivamento automático de eventos entregues (fora do escopo desta fase)
- Webhooks inbound

---

### 4. Fluxos detalhados e diagramas

**Fluxo principal — mudança de status com webhook inscrito**

1. Operador chama `PATCH /orders/:id/status` com novo status
2. `OrderService.changeStatus()` inicia transação SQL (`prisma.$transaction`)
3. Dentro da transação: valida transição de status, ajusta estoque se aplicável, atualiza `orders`, insere em `order_status_history`
4. Ainda dentro da mesma transação: chama `publishWebhookEvent(tx, order, fromStatus, toStatus)`
5. `publishWebhookEvent` consulta `webhook_endpoint` filtrando por `customer_id` e status de interesse; para cada endpoint inscrito, insere um registro em `webhook_outbox` com payload já renderizado (snapshot), `status = PENDING`, `event_id` UUID gerado neste momento
6. Transação commita; todos os registros são persistidos atomicamente
7. Worker (processo separado, polling de 2s) lê batch de eventos com `status = PENDING` ordenados por `created_at`
8. Para cada evento: busca configuração do endpoint (`webhook_endpoint`), assina payload com HMAC-SHA256 usando a secret ativa, realiza HTTP POST com headers obrigatórios, aguarda resposta com timeout de 10 segundos
9. Resposta 2xx: atualiza `webhook_outbox.status = DELIVERED`, registra em `webhook_delivery_log` com payload, status HTTP, tempo de resposta
10. Resposta não-2xx ou timeout: incrementa `retry_count`, calcula próximo `retry_at` conforme progressão de backoff, mantém `status = PENDING`

**Fluxo alternativo — rollback da transação**
- Se qualquer operação dentro da transação falhar (incluindo `publishWebhookEvent`), toda a transação dá rollback
- Nenhum evento é inserido na outbox; a mudança de status não ocorre
- Nenhuma notificação é disparada

**Fluxo alternativo — mudança de status sem webhook inscrito**
- `publishWebhookEvent` consulta `webhook_endpoint` e não encontra nenhum endpoint do cliente inscrito naquele status
- Nenhum registro é inserido na outbox
- Transação continua e commita normalmente

**Fluxo alternativo — esgotamento de retries e DLQ**
- Após a 5ª tentativa sem sucesso, worker move o evento para `webhook_dead_letter` com payload completo, `last_error` e `failed_at`
- `webhook_outbox` atualiza `status = FAILED`
- Operador ADMIN acessa `POST /admin/webhooks/dead-letter/:id/replay`
- Sistema insere novo registro em `webhook_outbox` com `status = PENDING` e registra auditoria em log

**Fluxo alternativo — rotação de secret**
- Cliente solicita rotação; sistema gera nova secret, armazena `secret_new` e `secret_old` com `secret_old_expires_at = now + 24h`
- Durante o grace period: worker verifica assinatura com ambas as secrets (tenta nova primeiro, depois antiga)
- Após `secret_old_expires_at`: `secret_old` é nulificado; apenas a nova secret é aceita

**Diagrama — ciclo de vida do evento na outbox**

```
[PENDING] --> (worker processa, sucesso)      --> [DELIVERED]
[PENDING] --> (worker processa, falha, < 5x)  --> [PENDING] (retry_at atualizado)
[PENDING] --> (worker processa, falha, 5x)    --> [FAILED] + [webhook_dead_letter]
[FAILED]  --> (replay admin)                  --> [PENDING] (novo registro outbox)
```

---

### 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)

---

**Endpoint 1: Cadastro de webhook**
- Tipo: http_endpoint
- Rota: `POST /webhooks`
- Autenticação: Bearer JWT obrigatório
- Semântica de status:
  - `201 Created` — endpoint criado com sucesso
  - `400 Bad Request` — URL inválida ou campos obrigatórios ausentes (`WEBHOOK_INVALID_URL`, `WEBHOOK_CUSTOMER_REQUIRED`)
  - `401 Unauthorized` — token ausente ou inválido

**Exemplo de requisição**
```json
{
  "customerId": "cust-uuid-123",
  "url": "https://erp.atlascomercial.com.br/webhooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
}
```

**Exemplo de resposta**
```json
{
  "data": {
    "id": "wh-uuid-456",
    "customerId": "cust-uuid-123",
    "url": "https://erp.atlascomercial.com.br/webhooks/oms",
    "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
    "secret": "whsec_a3f8c2d1e4b7...",
    "active": true,
    "createdAt": "2024-11-15T09:00:00Z"
  }
}
```

> A secret é retornada apenas neste momento. Não é possível recuperá-la depois; apenas rotacioná-la.

---

**Endpoint 2: Listagem de webhooks de um cliente**
- Tipo: http_endpoint
- Rota: `GET /webhooks?customerId=:customerId`
- Autenticação: Bearer JWT obrigatório
- Semântica de status:
  - `200 OK` — lista retornada (pode ser vazia)
  - `400 Bad Request` — `customerId` ausente

**Exemplo de resposta**
```json
{
  "data": [
    {
      "id": "wh-uuid-456",
      "customerId": "cust-uuid-123",
      "url": "https://erp.atlascomercial.com.br/webhooks/oms",
      "events": ["PAID", "SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2024-11-15T09:00:00Z"
    }
  ]
}
```

> A secret nunca é retornada em listagens.

---

**Endpoint 3: Edição de webhook**
- Tipo: http_endpoint
- Rota: `PATCH /webhooks/:id`
- Autenticação: Bearer JWT obrigatório
- Semântica de status:
  - `200 OK` — atualizado com sucesso
  - `400 Bad Request` — URL com HTTP ou campos inválidos
  - `404 Not Found` — webhook não encontrado (`WEBHOOK_NOT_FOUND`)

**Exemplo de requisição**
```json
{
  "url": "https://erp.atlascomercial.com.br/webhooks/oms-v2",
  "events": ["SHIPPED", "DELIVERED"]
}
```

**Exemplo de resposta**
```json
{
  "data": {
    "id": "wh-uuid-456",
    "url": "https://erp.atlascomercial.com.br/webhooks/oms-v2",
    "events": ["SHIPPED", "DELIVERED"],
    "active": true,
    "updatedAt": "2024-11-20T14:30:00Z"
  }
}
```

---

**Endpoint 4: Remoção de webhook**
- Tipo: http_endpoint
- Rota: `DELETE /webhooks/:id`
- Autenticação: Bearer JWT obrigatório
- Semântica de status:
  - `204 No Content` — removido com sucesso
  - `404 Not Found` — webhook não encontrado (`WEBHOOK_NOT_FOUND`)

---

**Endpoint 5: Histórico de entregas**
- Tipo: http_endpoint
- Rota: `GET /webhooks/:id/deliveries`
- Autenticação: Bearer JWT obrigatório
- Semântica de status:
  - `200 OK` — histórico retornado (pode ser vazia)
  - `404 Not Found` — webhook não encontrado (`WEBHOOK_NOT_FOUND`)

**Exemplo de resposta**
```json
{
  "data": [
    {
      "id": "del-uuid-789",
      "eventId": "evt-uuid-321",
      "status": "DELIVERED",
      "httpStatus": 200,
      "durationMs": 342,
      "attemptNumber": 1,
      "deliveredAt": "2024-11-15T09:00:02Z"
    },
    {
      "id": "del-uuid-790",
      "eventId": "evt-uuid-322",
      "status": "FAILED",
      "httpStatus": 503,
      "durationMs": 10001,
      "attemptNumber": 3,
      "nextRetryAt": "2024-11-15T09:30:00Z"
    }
  ]
}
```

---

**Endpoint 6: Replay de evento da DLQ (admin)**
- Tipo: http_endpoint
- Rota: `POST /admin/webhooks/dead-letter/:id/replay`
- Autenticação: Bearer JWT obrigatório + `requireRole('ADMIN')`
- Semântica de status:
  - `200 OK` — evento reinserido na outbox como PENDING
  - `403 Forbidden` — role insuficiente
  - `404 Not Found` — evento não encontrado na DLQ (`WEBHOOK_NOT_FOUND`)

**Exemplo de resposta**
```json
{
  "data": {
    "outboxId": "out-uuid-new",
    "originalDeadLetterId": "dl-uuid-999",
    "status": "PENDING",
    "replayedBy": "user-uuid-admin",
    "replayedAt": "2024-11-16T08:00:00Z"
  }
}
```

---

**Contrato 7: Payload enviado ao endpoint do cliente (saída do worker)**
- Tipo: http_endpoint (outbound)
- Método: `POST` para a URL cadastrada no webhook
- Headers enviados:

| Header | Valor | Descrição |
|---|---|---|
| `Content-Type` | `application/json` | Formato do payload |
| `X-Event-Id` | UUID v4 | Identificador único do evento para deduplicação |
| `X-Webhook-Id` | UUID v4 | Identificador do endpoint de webhook cadastrado |
| `X-Signature` | `sha256=<hmac_hex>` | HMAC-SHA256 do corpo da requisição usando a secret do endpoint |
| `X-Timestamp` | ISO 8601 | Timestamp do momento do disparo pelo worker |

**Exemplo de payload**
```json
{
  "eventId": "evt-uuid-321",
  "eventType": "order.status_changed",
  "timestamp": "2024-11-15T09:00:00.000Z",
  "orderId": "ord-uuid-abc",
  "orderNumber": "ORD-000042",
  "fromStatus": "PENDING",
  "toStatus": "PAID",
  "customerId": "cust-uuid-123",
  "totalCents": 15900
}
```

> Payload limitado a 64KB. Não inclui itens do pedido; cliente deve consultar `GET /orders/:id` se precisar do detalhamento.

---

### 6. Erros, exceções e fallback

**Matriz de erros do módulo de webhooks**

| Código | HTTP | Condição | Tratamento |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook ou evento não encontrado | Retorna 404 com mensagem |
| `WEBHOOK_INVALID_URL` | 400 | URL sem HTTPS ou formato inválido | Retorna 400 com campo e motivo |
| `WEBHOOK_CUSTOMER_REQUIRED` | 400 | `customerId` ausente no body | Retorna 400 com campo e motivo |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Secret ausente em operação que a exige | Retorna 400 com motivo |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload gerado excede 64KB | Registra erro em log; não insere na outbox |
| `WEBHOOK_DELIVERY_FAILED` | — | Resposta não-2xx ou timeout no disparo | Worker registra falha e agenda retry |
| `WEBHOOK_MAX_RETRIES_EXCEEDED` | — | 5ª tentativa falhou | Move para DLQ; atualiza outbox como FAILED |

**Estratégias de resiliência**
- **Timeout:** 10 segundos por chamada HTTP ao endpoint do cliente; sem timeout estendido
- **Retries:** 5 tentativas com backoff exponencial — 1min, 5min, 30min, 2h, 12h
- **Backoff:** tempo calculado a partir do `failed_at` do último registro de entrega
- **Sem circuit breaker** nesta fase: cliente com muitas falhas simplesmente acumula na DLQ; rate limiting e circuit breaker são pontos em aberto para fase futura

**Política de fallback**
- Evento que excede 64KB não é inserido na outbox; a mudança de status do pedido ocorre normalmente (a validação de tamanho ocorre antes da inserção)
- Evento na DLQ não é reprocessado automaticamente; requer intervenção manual via endpoint de replay

**Invariantes críticos**
- Um evento na `webhook_outbox` jamais pode existir sem uma mudança de status correspondente commitada no banco
- A secret nunca pode aparecer em logs (o logger Pino em `src/shared/logger/index.ts` já possui `redact` configurado para `*.token`; adicionar `*.secret` e `*.webhookSecret` à lista de redact)

---

### 7. Observabilidade

**Métricas**

- `webhook_outbox_size` — total de eventos por status (`PENDING`, `DELIVERED`, `FAILED`) na outbox; alerta se `PENDING` crescer acima de threshold configurável
- `webhook_delivery_duration_ms` — histograma de tempo de resposta dos endpoints dos clientes por `webhook_id`
- `webhook_delivery_success_total` — contador de entregas bem-sucedidas por `webhook_id`
- `webhook_delivery_failure_total` — contador de falhas por `webhook_id` e `attempt_number`
- `webhook_dlq_size` — total de eventos na `webhook_dead_letter`; alerta se crescer
- `webhook_worker_poll_duration_ms` — tempo de cada ciclo de polling do worker

**Logs**

- Formato: JSON estruturado via Pino (`src/shared/logger/index.ts`), campo `service: 'order-management-worker'` no processo worker
- Campos essenciais por evento processado:

```json
{
  "level": "info",
  "time": "2024-11-15T09:00:02.000Z",
  "service": "order-management-worker",
  "event": "webhook.delivery.success",
  "eventId": "evt-uuid-321",
  "webhookId": "wh-uuid-456",
  "customerId": "cust-uuid-123",
  "httpStatus": 200,
  "durationMs": 342,
  "attemptNumber": 1
}
```

- Campos em caso de falha: incluir `errorMessage`, `httpStatus` (ou `"timeout"`)
- Campos em caso de replay: incluir `replayedBy` (userId do operador ADMIN)
- A secret **nunca** deve aparecer em logs; adicionar `*.secret` e `*.webhookSecret` aos `redactPaths` do logger

**Tracing**

- Span `webhook.publish_event` — criado dentro da transação do `changeStatus`; atributos: `order_id`, `customer_id`, `from_status`, `to_status`, `webhooks_queued` (quantidade de registros inseridos)
- Span `webhook.worker.dispatch` — criado por evento processado pelo worker; atributos: `event_id`, `webhook_id`, `attempt_number`, `http_status`, `duration_ms`
- Amostragem: 100% para falhas, 10% para sucessos (configurável via variável de ambiente)

**Dashboards e alertas**

- Alerta: `webhook_dlq_size > 0` por mais de 5 minutos (indica eventos não entregues sem intervenção)
- Alerta: `webhook_delivery_failure_total` com taxa acima de 20% em janela de 5 minutos (indica problema sistêmico)
- Painel mínimo: tamanho da outbox por status ao longo do tempo + taxa de sucesso/falha por webhook

---

### 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
|---|---|---|
| Node.js | Versão em uso no projeto | Worker usa o mesmo runtime da API |
| Prisma Client | Versão em uso no projeto | Worker instancia `PrismaClient` próprio (mesmo banco, mesmo `DATABASE_URL`) |
| MySQL | Versão em uso no projeto | Novas tabelas adicionadas via migration Prisma |
| Pino | Versão em uso no projeto | Logger reaproveitado sem alteração; adicionar campos de redact |

**Garantias de compatibilidade**
- O módulo de webhooks segue exatamente a mesma estrutura dos módulos existentes: `controller.ts`, `service.ts`, `repository.ts`, `routes.ts`, `schemas.ts`
- Erros estendem `AppError` de `src/shared/errors/app-error.ts` — tratados automaticamente pelo `errorMiddleware` de `src/middlewares/error.middleware.ts` sem modificação
- O endpoint de replay usa `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` sem modificação

---

### 9. Critérios de aceite técnicos

- Mudança de status de pedido com webhook inscrito naquele status resulta em exatamente um registro por endpoint na `webhook_outbox` com `status = PENDING` e payload snapshot correto
- Rollback na transação de `changeStatus` não deixa nenhum registro na `webhook_outbox`
- Mudança de status sem webhook inscrito naquele status não gera nenhum registro na `webhook_outbox`
- Worker dispara HTTP POST em até 10 segundos após a inserção na outbox em condições normais
- Todas as chamadas do worker incluem os headers `X-Event-Id`, `X-Webhook-Id`, `X-Signature`, `X-Timestamp` e `Content-Type: application/json`
- Assinatura em `X-Signature` é HMAC-SHA256 do corpo da requisição com a secret do endpoint e pode ser verificada pelo cliente
- Resposta do cliente com timeout de 10 segundos ou status não-2xx é tratada como falha e agenda retry
- Progressão de retries segue exatamente: 1min, 5min, 30min, 2h, 12h
- Após 5 falhas: evento aparece em `webhook_dead_letter` com payload e `last_error`; `webhook_outbox` marca `status = FAILED`
- `POST /admin/webhooks/dead-letter/:id/replay` com role não-ADMIN retorna 403
- Replay gera novo registro na outbox com `status = PENDING` e linha de auditoria em log com `replayedBy`
- Durante grace period de rotação: assinatura com secret antiga e nova são ambas aceitas
- URL com HTTP é recusada com `WEBHOOK_INVALID_URL` em criação e edição
- Payload acima de 64KB retorna `WEBHOOK_PAYLOAD_TOO_LARGE`; mudança de status do pedido não é afetada
- Secret nunca aparece em resposta de listagem nem em logs

---

### 10. Riscos e mitigação

#### Bug na integração com `changeStatus` bloqueia mudanças de status em produção

- **Probabilidade:** baixa
- **Impacto:** a chamada a `publishWebhookEvent` dentro da transação pode causar rollback inadvertido de mudanças de status se lançar exceção inesperada, impactando diretamente a operação do OMS
- **Mitigação:**
  - Testar extensivamente o caminho de inserção na outbox junto com a transação de mudança de status antes do deploy
  - Cobrir com teste de integração: mudança de status bem-sucedida com e sem webhooks inscritos, e falha intencional na inserção da outbox
  - Revisão de código obrigatória antes do merge na branch principal
- **Plano de contingência:** remover temporariamente a chamada a `publishWebhookEvent` da transação via feature flag ou hotfix; processar eventos retroativos via replay da DLQ após correção

#### Acúmulo de eventos na outbox degrada performance do banco

- **Probabilidade:** baixa no curto prazo, média com crescimento de volume
- **Impacto:** queries do worker ficam lentas; latência de entrega supera 10 segundos
- **Mitigação:**
  - Garantir índice em `(status, created_at)` na `webhook_outbox` antes do go-live
  - Monitorar `webhook_outbox_size` e `webhook_worker_poll_duration_ms` desde o início
  - Definir política de arquivamento de eventos `DELIVERED` com mais de 30 dias (fora do escopo desta fase, mas deve ser planejado antes do crescimento do volume)
- **Plano de contingência:** aumentar tamanho do batch do worker ou reduzir intervalo de polling como ajuste operacional emergencial

#### Secret exposta em log ou resposta de API

- **Probabilidade:** baixa (requer bug de implementação)
- **Impacto:** terceiro com acesso aos logs pode forjar notificações para o endpoint do cliente
- **Mitigação:**
  - Adicionar `*.secret` e `*.webhookSecret` aos `redactPaths` do logger Pino em `src/shared/logger/index.ts`
  - Nunca retornar a secret em endpoints de listagem ou edição; apenas no `POST /webhooks` de criação
  - Revisão de segurança por Sofia antes do deploy, com verificação explícita de todos os pontos de log e resposta que tocam a secret
- **Plano de contingência:** rotação imediata de secret pelo cliente ao identificar exposição; invalidação da secret antiga após 24h

---

### 11. Integração com o sistema existente

Esta seção descreve como o módulo de webhooks se integra com componentes já existentes no código base, sem modificar a lógica interna deles.

**`src/modules/orders/order.service.ts` — extensão do método `changeStatus`**

Este é o único ponto onde o código existente precisa ser alterado. O método `changeStatus` (linha 126) executa uma transação SQL via `this.prisma.$transaction`. A integração consiste em adicionar uma chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro dessa transação, após a inserção em `order_status_history` e antes do `return`. A função recebe o client de transação ativo (`tx`) para garantir atomicidade — se a outbox falhar, a transação inteira dá rollback.

**`src/middlewares/auth.middleware.ts` — controle de acesso ao endpoint de replay**

O middleware exporta a função `requireRole(...roles: AuthUser['role'][])` (linha 49). O endpoint `POST /admin/webhooks/dead-letter/:id/replay` usará `requireRole('ADMIN')` exatamente como outros endpoints restritos da aplicação. Nenhuma alteração no middleware é necessária.

**`src/middlewares/error.middleware.ts` — tratamento automático de erros do módulo**

O `errorMiddleware` (linha 14) já trata qualquer instância de `AppError` convertendo-a para a resposta JSON padrão com `code`, `message` e `details`. Todos os erros do módulo de webhooks (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`) devem estender `AppError` de `src/shared/errors/app-error.ts` para serem tratados automaticamente, sem nenhuma modificação no middleware.

**`src/shared/logger/index.ts` — reuso e extensão do logger Pino**

O logger exportado (linha 32) é reutilizado pelo módulo de webhooks e pelo worker sem alteração de configuração, exceto por uma adição necessária: incluir `'*.secret'` e `'*.webhookSecret'` no array `redactPaths` (linha 4) para garantir que secrets nunca apareçam em logs, seguindo o mesmo padrão já aplicado a `*.password` e `*.token`.

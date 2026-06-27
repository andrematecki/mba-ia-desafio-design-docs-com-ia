# TRACKER — Rastreabilidade de Itens da Documentação

Cada linha mapeia um item registrado nos documentos à sua origem na transcrição da reunião ou no código fonte.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência entre mudança de status e disparo do webhook abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Reter os 3 clientes B2B que solicitaram a feature | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Reduzir chamadas de polling ao endpoint de listagem de pedidos | TRANSCRICAO | `[09:00] Marcos` |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastro de endpoint de webhook com URL, customer_id e lista de status | TRANSCRICAO | `[09:31] Marcos` |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | Secret gerada pelo sistema e retornada apenas na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Edição de webhook via PATCH | TRANSCRICAO | `[09:33] Bruno` |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Remoção de webhook via DELETE | TRANSCRICAO | `[09:33] Bruno` |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Listagem de webhooks por cliente via GET | TRANSCRICAO | `[09:33] Bruno` |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Disparo de HTTP POST ao endpoint do cliente quando status muda | TRANSCRICAO | `[09:03] Larissa` |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do payload no header X-Signature | TRANSCRICAO | `[09:20] Sofia` |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Identificador único X-Event-Id por evento para deduplicação | TRANSCRICAO | `[09:25] Diego` |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Retry automático com backoff exponencial em 5 tentativas | TRANSCRICAO | `[09:15] Diego` |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Evento movido para DLQ após esgotar tentativas | TRANSCRICAO | `[09:18] Diego` |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Histórico de entregas via GET /webhooks/:id/deliveries | TRANSCRICAO | `[09:34] Marcos` |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | Replay manual da DLQ via POST /admin/webhooks/dead-letter/:id/replay restrito a ADMIN | TRANSCRICAO | `[09:35] Diego` / `[09:36] Sofia` |
| PRD-RF-13 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24 horas | TRANSCRICAO | `[09:21] Sofia` |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência máxima de 10 segundos entre mudança de status e disparo | TRANSCRICAO | `[09:02] Marcos` |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | HTTPS obrigatório; URLs com HTTP recusadas na validação | TRANSCRICAO | `[09:23] Sofia` |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64KB por notificação | TRANSCRICAO | `[09:23] Sofia` / `[09:24] Diego` |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por chamada HTTP ao endpoint do cliente | TRANSCRICAO | `[09:42] Diego` |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Inserção na outbox atômica com a mudança de status | TRANSCRICAO | `[09:06] Diego` / `[09:41] Diego` |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Secret nunca exposta em logs | TRANSCRICAO | `[09:22] Diego` |
| PRD-ESC-01 | docs/PRD.md | Restrição (fora de escopo) | Notificação por e-mail em caso de falhas repetidas — adiado | TRANSCRICAO | `[09:37] Larissa` |
| PRD-ESC-02 | docs/PRD.md | Restrição (fora de escopo) | Rate limiting de envio por cliente — observar e implementar se necessário | TRANSCRICAO | `[09:39] Diego` |
| PRD-ESC-03 | docs/PRD.md | Restrição (fora de escopo) | Dashboard visual para clientes — projeto separado do frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-ESC-04 | docs/PRD.md | Restrição (fora de escopo) | Webhooks inbound — feature é exclusivamente outbound | TRANSCRICAO | `[09:02] Sofia` |
| PRD-ESC-05 | docs/PRD.md | Restrição (fora de escopo) | Garantia de ordenação global entre pedidos distintos — limitação aceita | TRANSCRICAO | `[09:13] Larissa` |
| PRD-DEC-01 | docs/PRD.md | Decisão | Outbox no MySQL em vez de disparo síncrono ou fila externa | TRANSCRICAO | `[09:06] Diego` / `[09:07] Larissa` |
| PRD-DEC-02 | docs/PRD.md | Decisão | At-least-once com X-Event-Id em vez de exactly-once | TRANSCRICAO | `[09:25] Diego` |
| PRD-DEC-03 | docs/PRD.md | Decisão | Chave secreta por endpoint em vez de chave global | TRANSCRICAO | `[09:21] Sofia` |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono descartado: travaria transação e impossibilitaria rollback | TRANSCRICAO | `[09:04] Bruno` / `[09:04] Larissa` |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Redis Streams descartado: overengineering para time pequeno | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-03 | docs/RFC.md | Trade-off | MySQL trigger descartado: não possui LISTEN/NOTIFY nativo | TRANSCRICAO | `[09:09] Diego` |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de envio por cliente: observar em produção | TRANSCRICAO | `[09:38] Diego` / `[09:39] Larissa` |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Escala horizontal do worker: particionamento por order_id adiado | TRANSCRICAO | `[09:13] Diego` |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | Comportamento quando lista de filtro de status está vazia: não definido | TRANSCRICAO | `[09:33] Marcos` |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL garante atomicidade sem nova infraestrutura | TRANSCRICAO | `[09:06] Diego` |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado.md | Decisão | Worker em processo separado com polling de 2 segundos | TRANSCRICAO | `[09:09] Diego` / `[09:11] Diego` |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | TRANSCRICAO | `[09:15] Diego` / `[09:17] Larissa` |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 com secret por endpoint e rotação com grace period | TRANSCRICAO | `[09:20] Sofia` / `[09:22] Sofia` |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-event-id.md | Decisão | Garantia at-least-once com X-Event-Id UUID para deduplicação pelo cliente | TRANSCRICAO | `[09:24] Diego` / `[09:26] Larissa` |
| ADR-006 | docs/adrs/ADR-006-webhook-no-monolito-existente.md | Decisão | Módulo de webhooks dentro do monólito existente em vez de microsserviço | TRANSCRICAO | `[09:27] Bruno` / `[09:28] Diego` |
| ADR-007 | docs/adrs/ADR-007-snapshot-payload-na-insercao.md | Decisão | Payload armazenado como snapshot na inserção na outbox | TRANSCRICAO | `[09:52] Larissa` |
| FDD-CONT-01 | docs/FDD.md | Contrato | POST /webhooks — cadastro de endpoint com URL, customerId e filtro de eventos | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONT-02 | docs/FDD.md | Contrato | GET /webhooks?customerId= — listagem de endpoints de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONT-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id — edição de endpoint | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONT-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id — remoção de endpoint | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONT-05 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONT-06 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay — replay restrito a ADMIN | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONT-07 | docs/FDD.md | Contrato | Payload outbound: event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONT-08 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | `[09:44] Diego` / `[09:44] Sofia` |
| FDD-ERRO-01 | docs/FDD.md | Restrição | WEBHOOK_INVALID_URL — URL sem HTTPS recusada com erro de validação | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-02 | docs/FDD.md | Restrição | WEBHOOK_NOT_FOUND — webhook ou evento não encontrado retorna 404 | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERRO-03 | docs/FDD.md | Restrição | WEBHOOK_PAYLOAD_TOO_LARGE — payload acima de 64KB resulta em erro | TRANSCRICAO | `[09:23] Sofia` / `[09:24] Diego` |
| FDD-ERRO-04 | docs/FDD.md | Restrição | WEBHOOK_MAX_RETRIES_EXCEEDED — evento movido para DLQ após 5 falhas | TRANSCRICAO | `[09:15] Diego` |
| FDD-ERRO-05 | docs/FDD.md | Restrição | Timeout de 10 segundos por chamada HTTP tratado como falha | TRANSCRICAO | `[09:42] Diego` |
| FDD-INT-01 | docs/FDD.md | Integração | Extensão do método changeStatus em order.service.ts para inserir evento na outbox dentro da transação | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | docs/FDD.md | Integração | Reuso de requireRole('ADMIN') do auth.middleware.ts no endpoint de replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-03 | docs/FDD.md | Integração | Erros do módulo estendem AppError de app-error.ts e são tratados pelo errorMiddleware automaticamente | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-04 | docs/FDD.md | Integração | Logger Pino de shared/logger/index.ts reutilizado; adicionar *.secret aos redactPaths | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-05 | docs/FDD.md | Integração | Padrão de código de erro WEBHOOK_* segue convenção INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Métrica webhook_outbox_size por status (PENDING, DELIVERED, FAILED) | TRANSCRICAO | `[09:08] Diego` |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Métrica webhook_dlq_size com alerta quando > 0 por mais de 5 minutos | TRANSCRICAO | `[09:18] Diego` |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Log estruturado por evento processado com eventId, webhookId, httpStatus, durationMs | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Auditoria de replay com replayedBy e timestamp | TRANSCRICAO | `[09:36] Sofia` |
| FDD-DEC-01 | docs/FDD.md | Decisão | Filtro de eventos aplicado na inserção na outbox, não no momento do envio | TRANSCRICAO | `[09:34] Bruno` / `[09:34] Diego` |
| FDD-DEC-02 | docs/FDD.md | Decisão | Payload sem itens do pedido para manter tamanho enxuto; cliente busca detalhes via GET /orders/:id | TRANSCRICAO | `[09:43] Diego` / `[09:44] Bruno` |
| FDD-DEC-03 | docs/FDD.md | Decisão | IDs da outbox em UUID seguindo padrão do restante do projeto | TRANSCRICAO | `[09:51] Larissa` |
| FDD-DEC-04 | docs/FDD.md | Decisão | Worker instancia PrismaClient próprio por ser processo separado | TRANSCRICAO | `[09:30] Bruno` |
| FDD-DEC-05 | docs/FDD.md | Decisão | customer_id passado no body da requisição; não derivado do JWT | TRANSCRICAO | `[09:32] Larissa` |

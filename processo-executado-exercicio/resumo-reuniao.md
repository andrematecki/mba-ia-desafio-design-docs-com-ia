# Resumo da Reunião — Sistema de Webhooks de Notificação de Pedidos

**Participantes:** Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Eng. Segurança)
**Duração:** ~55 minutos

---

## Por que a reunião aconteceu

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) faziam **polling** — chamavam `GET /orders` repetidamente para saber se algum pedido havia mudado. Isso é lento, caro e manual. A Atlas chegou a ameaçar migrar para o concorrente se a empresa não entregasse uma solução até o fim do trimestre.

A solução é inverter a lógica: **a empresa avisa o cliente quando algo muda**, em vez de o cliente ficar perguntando. Isso se chama **webhook** — o cliente registra uma URL e o sistema faz uma chamada HTTP quando o evento acontece.

O objetivo da reunião foi **decidir como construir esse sistema antes de começar a codar**.

---

## O fio da conversa

### 1. Síncrono ou assíncrono? `[09:03–09:08]`

A primeira pergunta foi: quando o status muda, o webhook é disparado na hora (dentro da mesma operação) ou de forma separada?

Bruno e Diego argumentaram que **síncrono é inviável**:
- A transação de mudança de status já é pesada (atualiza pedido, histórico e estoque)
- Se o cliente estiver offline, o que fazer? Dar rollback no pedido? Impossível.
- Um cliente lento travaria a operação para todos os outros pedidos

A solução escolhida foi o **padrão Outbox**: quando o status muda, dentro da mesma transação SQL, grava um registro numa tabela `webhook_outbox`. Um processo separado (worker) lê esses registros e dispara os webhooks. Se a transação principal falhar, o registro some junto — sem inconsistência possível.

### 2. Como o worker funciona? `[09:08–09:13]`

O worker roda em **loop a cada 2 segundos**, buscando eventos pendentes na tabela e disparando. Fica num processo Node.js separado da API (`src/worker.ts`), porque se a API reiniciar ele não pode morrer junto.

Bruno perguntou se triggers do banco resolveriam. Diego explicou que MySQL não tem LISTEN/NOTIFY como o Postgres, então polling de 2s é a solução mais simples e já atende o requisito dos clientes (abaixo de 10 segundos).

### 3. O que acontece se o cliente estiver fora do ar? `[09:14–09:18]`

Retry com **backoff exponencial**: tenta de novo após 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Total de 5 tentativas em ~15 horas. Se tudo falhar, o evento vai para a **DLQ (Dead Letter Queue)** — uma tabela separada `webhook_dead_letter` para guardar eventos não entregues.

Um endpoint admin (`POST /admin/webhooks/dead-letter/:id/replay`) permite reprocessar manualmente, exigindo role ADMIN.

### 4. Segurança: como o cliente sabe que a chamada veio da empresa? `[09:19–09:26]`

Sofia explicou que qualquer pessoa poderia fazer uma chamada HTTP para o endpoint do cliente fingindo ser o sistema. A solução é **HMAC-SHA256**: o payload é assinado com uma chave secreta compartilhada. O cliente verifica a assinatura antes de processar.

Cada endpoint de webhook tem sua própria chave secreta — não uma global — para que se vazar uma, não comprometa todos os clientes. A chave pode ser rotacionada com 24h de grace period para o cliente migrar seus sistemas.

Também ficou decidido: HTTPS obrigatório, limite de 64KB por payload.

### 5. E se o evento chegar duas vezes? `[09:24–09:26]`

O sistema garante **at-least-once** (pelo menos uma entrega), não exactly-once. É possível o cliente receber o mesmo evento duplicado — por exemplo, se o worker disparou mas não recebeu confirmação de entrega antes de reiniciar.

A solução: cada evento tem um `X-Event-Id` único (UUID). O cliente usa esse ID para detectar duplicatas do lado dele. É o mesmo padrão usado por Stripe e GitHub.

### 6. Como integrar com o código existente? `[09:27–09:41]`

Bruno propôs seguir o mesmo padrão de módulos do projeto: criar `src/modules/webhooks/` com controller, service, repository, routes e schemas. Os códigos de erro vão seguir o prefixo `WEBHOOK_`. Logger Pino e error middleware já existem e não precisam de alteração.

O ponto mais crítico: a função `publishWebhookEvent(tx, order, fromStatus, toStatus)` precisa ser chamada **dentro da transação** do `changeStatus` em `src/modules/orders/order.service.ts` — senão perde a garantia de consistência atômica.

### 7. O que ficou de fora `[09:37–09:40]`

- **Email de alerta** quando webhook falha repetidamente → adiado para próxima fase
- **Rate limiting** de envio → observar e implementar se virar problema
- **Dashboard visual** → projeto separado do time de frontend
- **Garantia de ordenação global** de eventos → limitação conhecida, aceita enquanto houver single-worker

---

## Decisões fechadas

| Decisão | O que foi escolhido | Por que descartaram as alternativas |
|---|---|---|
| Outbox no MySQL | Inserir evento na mesma transação do `changeStatus` | Síncrono travaria outros pedidos; Redis é overengineering para equipe pequena |
| Worker separado, polling 2s | Processo Node.js independente em `src/worker.ts` | MySQL não tem LISTEN/NOTIFY; 2s atende o requisito de <10s |
| 5 tentativas com backoff 1m/5m/30m/2h/12h | Janela de ~15h de retry | 3 tentativas cobriria apenas ~30min — insuficiente para manutenção planejada |
| DLQ em tabela separada | Tabela `webhook_dead_letter` | Mais limpa que marcar "failed" na outbox; facilita debug e reprocessamento |
| HMAC-SHA256, secret por endpoint | Assinatura do payload com chave individual | Secret global compromete todos se vazar; HMAC-SHA256 é padrão de mercado |
| At-least-once + X-Event-Id | Cliente deduplica pelo header UUID | Exactly-once exige coordenação dos dois lados, muito mais complexo |
| Snapshot do payload na inserção | Payload renderizado quando entra na outbox | Se o pedido mudar depois, o evento ainda reflete o estado correto na época |
| UUID para IDs da outbox | Seguir padrão do restante do projeto | Consistência — todo o projeto já usa UUID |

---

## Requisitos funcionais identificados

1. `POST /webhooks` — cadastrar endpoint de webhook (url, filtro de eventos, customer_id)
2. `PATCH /webhooks/:id` — editar configuração
3. `DELETE /webhooks/:id` — remover webhook
4. `GET /webhooks?customerId=` — listar webhooks de um cliente
5. `GET /webhooks/:id/deliveries` — histórico de entregas (sucesso/falha, payload, response, tempo)
6. `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar evento da DLQ (role ADMIN)
7. Filtro de eventos por status na inserção na outbox (não no envio)
8. Rotação de secret com grace period de 24h
9. Auditoria do replay (log de quem acionou)

---

## Formato do payload enviado ao cliente

```json
{
  "event_id": "uuid",
  "event_type": "order.status_changed",
  "timestamp": "2024-11-15T09:00:00Z",
  "order_id": "uuid",
  "order_number": "ORD-000042",
  "from_status": "PENDING",
  "to_status": "PAID",
  "customer_id": "uuid",
  "total_cents": 15900
}
```

## Headers enviados ao cliente

| Header | Conteúdo |
|---|---|
| `X-Event-Id` | UUID único do evento (para deduplicação) |
| `X-Signature` | HMAC-SHA256 do payload |
| `X-Timestamp` | Timestamp do envio (para detectar replay attack) |
| `X-Webhook-Id` | ID do endpoint webhook cadastrado |
| `Content-Type` | `application/json` |

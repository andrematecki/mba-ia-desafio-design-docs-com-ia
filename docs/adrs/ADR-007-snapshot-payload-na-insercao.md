# ADR-007 — Payload do evento armazenado como snapshot no momento da inserção

## Status
Aceito

## Contexto

Ao inserir um evento na `webhook_outbox`, é necessário decidir o que armazenar: o payload completo já renderizado com os dados do pedido naquele momento (snapshot), ou apenas uma referência ao pedido (ex: `order_id`) para renderizar o payload no momento do envio.

Essa decisão afeta a consistência dos dados entregues ao cliente e o comportamento em cenários onde o pedido é modificado após a geração do evento mas antes da entrega.

## Decisão

Armazenar o **payload completo já renderizado no momento da inserção na outbox** (snapshot). O worker usa esse payload diretamente no disparo, sem consultar o estado atual do pedido.

## Alternativas Consideradas

### Armazenar apenas referência (order_id) e renderizar no momento do envio
Guardar apenas o identificador do pedido na outbox e construir o payload no momento em que o worker for disparar o webhook.

**Por que foi descartada:** se o pedido sofrer alterações entre a geração do evento e o momento do envio (especialmente em cenários de retry, onde o intervalo pode ser de horas), o payload entregue ao cliente não refletiria o estado do pedido quando o status mudou — refletiria um estado posterior. Isso viola a semântica do evento: o cliente espera receber os dados do momento da mudança, não o estado atual. Além disso, consultas ao banco no momento do envio aumentam a carga desnecessariamente.

## Consequências

**Positivas**
- O payload entregue ao cliente representa fielmente o estado do pedido no momento em que a mudança de status ocorreu
- Worker não precisa fazer consultas adicionais ao banco no momento do disparo — o payload já está pronto para envio
- Comportamento consistente em todos os retries: o mesmo payload é reenviado em todas as tentativas, sem variação

**Negativas**
- Maior consumo de espaço em banco por armazenar payloads completos em vez de apenas referências
- Se o schema do payload precisar mudar no futuro, eventos antigos ainda na outbox ou na DLQ terão o formato anterior

**Trade-off explícito:** fidelidade semântica do evento e simplificação do worker em troca de maior uso de espaço em banco.

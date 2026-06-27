# ADR-001 — Padrão Outbox no MySQL para geração de eventos de webhook

## Status
Aceito

## Contexto

Quando o status de um pedido muda, o sistema precisa gerar uma notificação para os endpoints de webhook cadastrados pelos clientes. O desafio é garantir que a notificação seja gerada se e somente se a mudança de status ocorrer — sem inconsistência entre os dois.

A transação de mudança de status já é composta por múltiplas operações: atualização do pedido, inserção no histórico de status e ajuste de estoque. Qualquer mecanismo de notificação precisa ser integrado a esse contexto transacional.

Três abordagens foram avaliadas:
1. Disparo síncrono do webhook dentro da própria transação de mudança de status
2. Publicação em fila de mensagens externa (Redis Streams, RabbitMQ, Kafka)
3. Padrão Outbox: inserção de evento em tabela auxiliar dentro da mesma transação

## Decisão

Adotar o **padrão Outbox sobre o MySQL existente**: ao executar a mudança de status, inserir um registro na tabela `webhook_outbox` dentro da mesma transação SQL. Um worker separado lê essa tabela e realiza os disparos HTTP de forma assíncrona.

## Alternativas Consideradas

### Disparo síncrono dentro da transação
Realizar a chamada HTTP ao endpoint do cliente dentro da transação de mudança de status.

**Por que foi descartada:** a transação ficaria bloqueada aguardando resposta de um sistema externo. Se o cliente estiver offline, lento ou retornar erro, a mudança de status do pedido seria impedida ou exigiria rollback — impactando diretamente a operação. Além disso, o tempo de resposta da operação de negócio passaria a depender da latência de cada cliente.

### Fila de mensagens externa (Redis Streams, RabbitMQ, Kafka)
Publicar o evento em uma fila externa no momento da mudança de status, com worker consumindo dessa fila.

**Por que foi descartada:** introduz nova infraestrutura que o time precisaria provisionar, operar e monitorar. Para o volume atual e a complexidade do problema, o overhead não se justifica. O padrão Outbox resolve o mesmo problema de desacoplamento usando apenas o MySQL já existente.

## Consequências

**Positivas**
- Consistência atômica garantida: se a transação de mudança de status der rollback, o evento some junto; se commitar, o evento está registrado
- Sem nova infraestrutura: utiliza o banco de dados já existente e operado pelo time
- Desacoplamento total entre a operação de negócio e a entrega do webhook: cliente offline não impacta mudanças de status

**Negativas**
- O banco de dados assume papel de fila, o que pode gerar pressão em volumes muito altos de eventos
- Latência mínima de entrega determinada pelo intervalo de polling do worker (não é push imediato)
- Requer gestão do ciclo de vida da tabela de outbox (arquivamento de eventos antigos, índices adequados)

**Trade-off explícito:** simplicidade operacional e consistência forte em troca de latência mínima de polling e uso do banco como fila.

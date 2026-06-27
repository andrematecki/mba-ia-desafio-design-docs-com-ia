# ADR-002 — Worker de disparo de webhooks em processo separado com polling

## Status
Aceito

## Contexto

Definido o padrão Outbox (ADR-001), é necessário decidir como o worker que lê a fila e realiza os disparos HTTP vai operar: onde roda, como é ativado e com que frequência verifica a fila.

Duas questões precisavam ser respondidas:
1. O worker roda no mesmo processo da API ou em processo separado?
2. O worker age por polling periódico ou é ativado por algum mecanismo de notificação do banco?

## Decisão

O worker roda como **processo Node.js separado** da API, com **polling a cada 2 segundos** na tabela `webhook_outbox`. Terá seu próprio entry-point (`src/worker.ts`) e será iniciado com script independente (`npm run worker`). Usa o mesmo banco de dados via nova instância do Prisma Client, já que cada processo Node.js mantém sua própria instância.

## Alternativas Consideradas

### Worker no mesmo processo da API
Rodar o worker como um loop em background dentro do processo da API (ex: `setInterval` na inicialização).

**Por que foi descartada:** se a API reiniciar por deploy, crash ou scaling, o worker seria interrompido junto, potencialmente no meio de um ciclo de processamento. Processos separados permitem ciclos de vida independentes: a API pode ser redeploy'd sem interromper entregas em andamento.

### Trigger no banco de dados para ativar o worker reativamente
Usar trigger no MySQL para detectar inserções na outbox e notificar o worker, eliminando o polling.

**Por que foi descartada:** o MySQL não possui mecanismo nativo de notificação para processos externos equivalente ao `LISTEN/NOTIFY` do PostgreSQL. Qualquer implementação exigiria improviso — escrever em arquivo, bater em um endpoint — adicionando complexidade e pontos de falha sem ganho real. Polling a cada 2 segundos é suficiente para atender o requisito de latência abaixo de 10 segundos.

## Consequências

**Positivas**
- Ciclos de vida independentes: reinicializações da API não interrompem o processamento da fila
- Separação de responsabilidades clara entre processo de API (requisições) e worker (entregas)
- Polling de 2 segundos atende o requisito de notificação abaixo de 10 segundos com margem

**Negativas**
- Requer gerenciamento de dois processos em produção (API e worker)
- Worker consome conexão de banco continuamente, mesmo em períodos sem eventos
- Ordenação global de eventos entre diferentes pedidos não é garantida se o worker for escalado horizontalmente no futuro (limitação conhecida e aceita para o escopo atual)

**Trade-off explícito:** resiliência e separação de responsabilidades em troca de complexidade operacional adicional (dois processos para gerenciar).

# ADR-003 — Política de retry com backoff exponencial e Dead Letter Queue

## Status
Aceito

## Contexto

O worker realiza chamadas HTTP para endpoints externos de clientes. Esses endpoints podem estar temporariamente indisponíveis por falha, manutenção planejada ou sobrecarga. É necessário definir o que acontece quando uma entrega falha: quantas vezes tentar novamente, com qual intervalo entre tentativas e o que fazer quando todas as tentativas se esgotam.

Duas variáveis precisavam ser decididas:
1. Quantas tentativas e com que progressão de intervalos?
2. Para onde vai o evento após esgotar as tentativas?

## Decisão

**5 tentativas** com backoff exponencial na progressão **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas**, cobrindo uma janela total de aproximadamente 15 horas entre a primeira falha e a última tentativa.

Após esgotar as 5 tentativas sem sucesso, o evento é movido para uma **Dead Letter Queue (DLQ)** persistida em tabela separada (`webhook_dead_letter`), com registro de payload, motivo da falha e timestamp. Reprocessamento manual via endpoint admin restrito a operadores com perfil de administrador.

## Alternativas Consideradas

### 3 tentativas com backoff agressivo
Reduzir para 3 tentativas, tornando o sistema mais agressivo em considerar falhas permanentes.

**Por que foi descartada:** 3 tentativas com backoff razoável cobriria uma janela de apenas 30 a 40 minutos. Um cliente com manutenção planejada de 2 horas perderia todos os eventos gerados nesse período para a DLQ sem possibilidade de entrega automática. O time já havia vivenciado esse cenário com clientes reais.

### Retry indefinido com backoff crescente
Tentar indefinidamente, com intervalos cada vez maiores.

**Por que foi descartada:** evento pendente indefinidamente dificulta o rastreamento operacional e pode mascarar clientes efetivamente descontinuados. Um teto de tentativas garante que eventos problemáticos sejam identificados e tratados explicitamente via DLQ.

### Marcar como falha na própria tabela de outbox (sem DLQ separada)
Manter os eventos falhados na mesma tabela `webhook_outbox` com status diferente, sem tabela separada.

**Por que foi descartada:** misturar eventos em processamento com eventos falhados definitivamente polui as queries do worker e dificulta o debug. Tabela separada para DLQ mantém o fluxo principal limpo e facilita consultas de evidência para reprocessamento.

## Consequências

**Positivas**
- Janela de ~15 horas cobre a maioria das manutenções planejadas de clientes sem perda de eventos
- DLQ em tabela separada facilita diagnóstico e reprocessamento sem poluir o fluxo principal
- Endpoint de replay manual permite recuperação controlada após incidentes

**Negativas**
- Eventos podem permanecer em processamento por até ~15 horas antes de ir para a DLQ
- Clientes com indisponibilidade superior a 15 horas perdem a entrega automática e dependem de intervenção manual
- Requer gerenciamento operacional da DLQ (monitoramento, replay periódico)

**Trade-off explícito:** cobertura de indisponibilidades prolongadas em troca de tempo de espera maior antes do descarte para DLQ.

# ADR-005 — Garantia de entrega at-least-once com identificador único por evento

## Status
Aceito

## Contexto

Em sistemas distribuídos com retry, é possível que o worker dispare uma chamada ao cliente, o cliente processe e responda com sucesso, mas o worker não receba a confirmação (timeout de rede, crash antes de atualizar o status na outbox). Nesse cenário, o worker tentará novamente e o cliente receberá o mesmo evento duas vezes.

É necessário definir a semântica de entrega: o sistema deve garantir que cada evento chegue exatamente uma vez (exactly-once) ou pelo menos uma vez (at-least-once)?

## Decisão

Adotar semântica **at-least-once**: o sistema garante que cada evento será entregue ao menos uma vez, mas pode entregar o mesmo evento mais de uma vez em cenários de falha do worker após o disparo mas antes do registro de confirmação.

Para permitir que o cliente lide com duplicatas, cada evento recebe um **identificador único (UUID)** gerado no momento da inserção na outbox, enviado ao cliente no header `X-Event-Id`. O cliente é responsável por deduplicar usando esse identificador.

## Alternativas Consideradas

### Garantia exactly-once
Garantir que cada evento seja processado pelo cliente exatamente uma vez, sem duplicatas possíveis.

**Por que foi descartada:** exactly-once exige coordenação entre os dois lados da comunicação — o sistema precisaria armazenar confirmação de recebimento do cliente e o cliente precisaria expor algum mecanismo de idempotência verificável pelo sistema. Isso aumenta significativamente a complexidade da integração, do protocolo e da implementação. At-least-once com `X-Event-Id` resolve 99% dos casos práticos com fração da complexidade. É o padrão adotado por Stripe e GitHub.

### At-least-once sem identificador de evento
Garantir pelo menos uma entrega, mas sem enviar identificador único para o cliente.

**Por que foi descartada:** sem identificador, o cliente não tem como diferenciar uma segunda entrega legítima de um evento diferente de uma duplicata. O `X-Event-Id` é o mecanismo mínimo necessário para que o cliente possa implementar deduplicação de forma confiável.

## Consequências

**Positivas**
- Implementação significativamente mais simples que exactly-once
- Padrão amplamente reconhecido e documentado — clientes familiarizados com Stripe ou GitHub já conhecem o modelo
- `X-Event-Id` único por evento permite deduplicação confiável do lado do cliente
- Sem necessidade de coordenação de estado entre sistema e cliente além do identificador

**Negativas**
- Responsabilidade de deduplicação transferida para o cliente — aumenta a complexidade da integração do lado dele
- Cliente que não implementar deduplicação pode processar eventos duplicados com efeitos colaterais
- Requer documentação clara do contrato no portal de desenvolvedores para que clientes entendam a semântica

**Trade-off explícito:** simplicidade de implementação no sistema em troca de maior responsabilidade do lado do cliente.

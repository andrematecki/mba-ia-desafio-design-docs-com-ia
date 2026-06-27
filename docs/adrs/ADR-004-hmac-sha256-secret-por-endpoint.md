# ADR-004 — Autenticação HMAC-SHA256 com chave secreta por endpoint

## Status
Aceito

## Contexto

O worker realiza chamadas HTTP POST para endpoints externos de clientes com dados de pedidos. O cliente precisa conseguir verificar que a chamada recebida veio realmente do sistema e que o payload não foi adulterado em trânsito. Sem mecanismo de autenticação, qualquer agente externo poderia forjar notificações para o endpoint do cliente.

Além disso, é necessário decidir o escopo da chave secreta: uma chave global compartilhada entre todos os endpoints, ou uma chave por endpoint cadastrado.

## Decisão

Assinar cada payload com **HMAC-SHA256** usando uma **chave secreta única por endpoint**. A assinatura é enviada no header `X-Signature`. A chave é gerada pelo sistema no momento do cadastro do endpoint e entregue ao cliente apenas nesse momento. O cliente pode solicitar rotação da chave a qualquer momento, com grace period de 24 horas durante o qual ambas as chaves (antiga e nova) são aceitas.

HTTPS é obrigatório para todos os endpoints cadastrados — URLs com HTTP são recusadas na validação.

## Alternativas Consideradas

### Chave secreta global compartilhada entre todos os endpoints
Usar uma única chave secreta da plataforma, compartilhada entre todos os clientes para verificação de autenticidade.

**Por que foi descartada:** uma chave global comprometida afeta todos os clientes simultaneamente. Um precedente real foi citado na reunião: um cliente anterior havia vazado a chave em log de aplicação. Com chave por endpoint, o raio de impacto de um vazamento é limitado a um único endpoint.

### Sem assinatura — verificação apenas por HTTPS e IP de origem
Confiar exclusivamente no TLS para garantir autenticidade, sem assinatura de payload.

**Por que foi descartada:** TLS garante confidencialidade e integridade do canal, mas não permite que o cliente verifique a identidade do remetente de forma independente. IP de origem é frágil em ambientes com NAT, proxies ou mudanças de infraestrutura. HMAC permite verificação criptográfica do payload mesmo sem dependência de rede.

## Consequências

**Positivas**
- Cliente consegue verificar criptograficamente que a notificação veio do sistema e que o payload não foi adulterado
- Vazamento de uma chave impacta apenas o endpoint comprometido, não toda a plataforma
- HMAC-SHA256 é padrão de mercado com ampla disponibilidade de bibliotecas em qualquer linguagem
- Rotação com grace period de 24 horas permite troca de chave sem interrupção do serviço do cliente
- Header `X-Timestamp` permite que o cliente detecte replay attacks se necessário

**Negativas**
- Gerenciamento de múltiplas chaves (uma por endpoint cadastrado) em vez de uma única chave global
- Cliente precisa implementar verificação de assinatura do lado dele — aumenta a complexidade da integração
- Rotação de chave requer coordenação entre o sistema e o cliente dentro da janela de 24 horas

**Trade-off explícito:** isolamento de segurança por endpoint em troca de maior complexidade de gerenciamento de chaves.

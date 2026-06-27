# ADR-006 — Módulo de webhooks implementado dentro do monólito existente

## Status
Aceito

## Contexto

O OMS é um monólito Node.js com módulos bem definidos em `src/modules/`. Ao adicionar a feature de webhooks, é necessário decidir onde ela vive: dentro do monólito existente como mais um módulo, ou como um serviço separado com deploy e ciclo de vida independentes.

O time é pequeno. A feature precisa ser entregue em três sprints. Não há infraestrutura de orquestração de containers em uso.

## Decisão

O sistema de webhooks será implementado **dentro do monólito existente**, como novo módulo `src/modules/webhooks/` seguindo a mesma estrutura dos demais módulos. O worker de disparo será um **segundo processo Node.js** (`src/worker.ts`) que compartilha o mesmo repositório e banco de dados, mas roda com ciclo de vida separado — não um serviço independente com deploy próprio.

## Alternativas Consideradas

### Implementar como microsserviço separado
Criar um serviço independente responsável por webhooks, com seu próprio repositório, deploy, banco de dados e API interna de comunicação com o OMS.

**Por que foi descartada:** introduziria sobrecarga operacional desproporcional ao tamanho do time e ao escopo da feature — CI/CD separado, monitoramento adicional, comunicação entre serviços, eventual consistência entre bancos. O time optou por resolver o problema de desacoplamento do disparo com um worker em processo separado dentro do mesmo monólito, sem os custos de um microsserviço. A complexidade adicionada não se justificaria no estágio atual.

### Implementar como Lambda / função serverless
Disparar os webhooks a partir de função serverless ativada por eventos da outbox.

**Por que foi descartada:** exigiria mudança de infraestrutura que o time não opera hoje. A outbox no MySQL não tem mecanismo nativo de trigger para funções externas — seria necessário polling de qualquer forma, ou introduzir um broker intermediário. Não resolve o problema com menos complexidade que o worker em processo.

## Consequências

**Positivas**
- Desenvolvimento mais rápido: mesmo repositório, mesmas ferramentas, mesmos padrões de código, mesma pipeline de CI
- Worker compartilha banco de dados e variáveis de ambiente sem necessidade de configuração adicional
- Revisão de código acontece no mesmo fluxo do restante do projeto
- Rollback de feature é simples — é código no mesmo repositório

**Negativas**
- Worker e API compartilham o mesmo processo de build e repositório — escalar ou deployar um implica o outro
- Se o monólito crescer e a decisão de extrair webhooks como serviço for tomada no futuro, há custo de migração
- Falha catastrófica do servidor afeta tanto a API quanto o worker (diferente de serviços com deploys independentes)

**Trade-off explícito:** velocidade de entrega e simplicidade operacional hoje em troca de maior esforço de eventual extração futura caso o módulo precise escalar de forma independente.

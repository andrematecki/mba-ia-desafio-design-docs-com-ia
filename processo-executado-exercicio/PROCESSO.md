# Processo de Produção — Design Docs com IA

## Sobre o desafio

O desafio consistiu em transformar a transcrição de uma reunião técnica de 55 minutos em um pacote completo de documentação de engenharia para uma feature de Sistema de Webhooks de Notificação de Pedidos. O ponto de partida era somente o arquivo `TRANSCRICAO.md` e o código de uma aplicação OMS em Node.js + TypeScript. A entrega esperada era um conjunto de documentos — PRD, RFC, FDD, ADRs e Tracker — acionável o suficiente para um time de engenharia iniciar a implementação sem depender da memória de quem participou da reunião.

O principal desafio não foi gerar os documentos, mas garantir que cada um operasse na altitude correta: o PRD respondendo "por que e o quê" sem vazar detalhes de implementação, a RFC propondo a arquitetura sem descer ao nível do FDD, os ADRs registrando apenas decisões genuinamente arquiteturais e não convenções de código, e o FDD sendo técnico o suficiente para um desenvolvedor pegar e começar a implementar. Essa separação foi o ponto que exigiu mais revisão e julgamento ao longo do processo.

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
|---|---|
| **Claude Code** | Ferramenta principal. Leu o código fonte, analisou a transcrição, gerou todos os documentos finais e iterou sobre eles conforme feedback. Também foi usado para comparar os resultados das outras ferramentas. |
| **Claude.ai** (com prompt da faculdade) | Gerou uma versão alternativa do PRD via prompt de entrevista estruturada. Resultado usado como referência de formato e profundidade para calibrar o documento final. |
| **ChatGPT** (com prompt da faculdade) | Gerou outra versão alternativa do PRD. Resultado usado na comparação: seguiu o formato correto mas produziu conteúdo raso, evidenciando que o processo de entrevista importa mais que a ferramenta. |

---

## Workflow adotado

### Ordem de produção

O desafio recomenda a ordem ADRs → RFC → FDD → PRD. Optei por uma ordem diferente por razões pedagógicas:

1. **Leitura e mapeamento** — Antes de gerar qualquer documento, li a transcrição inteira e mapeei o código existente (`order.service.ts`, middlewares, padrão de erros, logger). Esse mapeamento alimentou todos os documentos e evitou que a IA inventasse referências de código inexistentes.

2. **PRD** — Gerado primeiro para entender o escopo de produto antes de descer ao nível técnico. Gerei três versões (Claude Code, Claude.ai e ChatGPT) e comparei os resultados antes de produzir a versão final.

3. **RFC** — Com o escopo de produto definido, a RFC consolidou a proposta técnica. Os links para os ADRs foram adicionados antecipadamente, antes dos ADRs existirem.

4. **ADRs** — Gerados em paralelo (7 documentos simultâneos). Depois revisados contra os critérios formais de quando uma ADR faz sentido existir.

5. **FDD** — Com PRD, RFC e ADRs prontos, o FDD foi gerado seguindo o template do prompt da faculdade. É o documento mais técnico e o único que referencia arquivos reais do código com números de linha.

6. **TRACKER** — Gerado por último, varrendo todos os documentos produzidos e mapeando cada item à sua origem na transcrição (com timestamp) ou no código (com caminho de arquivo).

### Como a IA foi usada

Em vez de prompts genéricos ("gere um PRD a partir desta transcrição"), o processo foi conduzido em etapas menores com instruções específicas: primeiro extrair e organizar o que foi discutido, depois gerar cada seção com base nesse entendimento, depois revisar e corrigir. A IA foi usada como executora, não como definidora do que deveria ser produzido.

---

## Prompts customizados

### Prompt 1 — Resumo executivo de reunião (uso geral)

Criado durante o processo para extrair informações estruturadas de qualquer transcrição de reunião — não apenas de reuniões técnicas. Adaptável a syncs de indicadores, alinhamentos e reuniões de pessoas.

```
Você vai receber a transcrição de uma reunião. Sua tarefa é extrair
as informações mais importantes e organizar em um resumo executivo.

Antes de escrever, identifique o tipo de reunião (definição técnica,
alinhamento, sync de status, revisão de indicadores, feedback/pessoas,
ou outro) e adapte as seções ao que faz sentido para esse tipo.
Omita seções que não se aplicam. Não force estrutura onde não há conteúdo.

**Seções disponíveis (use as que fizerem sentido):**

## Contexto e objetivo
Por que essa reunião aconteceu? Qual era o problema ou decisão a tomar?

## O que foi discutido
Tópicos principais em ordem cronológica, com 1-2 frases por tópico
explicando o raciocínio — não o que foi dito, mas o que foi debatido e por quê.

## Decisões tomadas
*(só se houver decisões efetivas)*
Tabela: Decisão | O que foi escolhido | Por que as alternativas foram descartadas

## Status e indicadores
*(só para reuniões de acompanhamento)*
O que está no caminho certo, o que está em risco, o que está atrasado.

## O que ficou de fora ou adiado
Itens explicitamente descartados ou postergados, com o motivo.

## Próximos passos e responsáveis
Ações com dono e prazo. Se prazo não foi mencionado: "a definir".

## Perguntas em aberto
Pontos levantados mas não resolvidos.

---

**Regras:**
- Não invente nada. Se não foi mencionado, não inclua.
- Se algo foi descartado, não coloque como próximo passo.
- Use linguagem direta. Sem frases genéricas de abertura.
- Ignore ruído da transcrição (fala cruzada, frases incompletas).

**Transcrição:**
[cole aqui]
```

### Prompt 2 — Revisão de ADRs contra critério formal

Usado para revisar os ADRs gerados e eliminar os que não se sustentavam como decisões arquiteturais reais.

```
Revise cada ADR abaixo avaliando se ela faz sentido existir como ADR,
usando os critérios abaixo:

**Use ADR quando:**
- Decisão de tecnologia, framework ou biblioteca com impacto duradouro
- Mudança arquitetural que afeta múltiplos times ou sistemas
- Trade-off explícito entre alternativas relevantes
- Decisão que será questionada no futuro e você quer a resposta documentada

**Não use ADR quando:**
- A decisão é de implementação de baixo nível (vai no código ou no FDD)
- Não há alternativa real que tenha sido considerada e descartada
- O overhead de manutenção supera o benefício

Para cada ADR, responda:
1. Ela documenta uma decisão arquitetural genuína ou uma convenção de implementação?
2. Qual é a alternativa real que foi descartada?
3. Essa decisão será questionada no futuro? Por quem e em que contexto?
4. Veredito: manter, reescrever com foco diferente, ou remover e mover para o FDD.
```

---

## Iterações e ajustes

### Iteração 1 — PRD gerado três vezes antes da versão final

O PRD foi o primeiro documento gerado e passou por três versões antes de chegar ao resultado final.

A versão do **ChatGPT** (usando o prompt da faculdade) seguiu o formato correto mas produziu conteúdo raso: requisitos funcionais eram apenas uma lista de nomes sem fluxo principal, métricas tinham "não definido na reunião" em campos que a própria transcrição respondia, critérios de aceitação eram vagos ("CRUD funcional"). O formato estava certo, o conteúdo não.

A versão do **Claude.ai** (mesmo prompt) foi melhor em conteúdo mas vazou detalhes de implementação que pertencem ao FDD: nomes de funções (`publishWebhookEvent(tx, order, fromStatus, toStatus)`), caminhos de arquivo (`src/modules/webhooks/`), detalhes de instância do Prisma. Um PRD deve ser legível por um PM não-técnico — se uma frase exige conhecimento de código para ser entendida, ela está no documento errado.

A versão final combinou o formato do template da faculdade com o conteúdo rico de ambas as versões, removendo tudo que era nível de implementação.

### Iteração 2 — ADR-006 reescrita do zero

A primeira versão da ADR-006 registrava "reuso dos padrões existentes do projeto" como decisão arquitetural. Após revisão contra os critérios formais, ficou claro que isso era uma convenção de implementação, não uma decisão arquitetural: a "alternativa" (criar padrão próprio) não foi cogitada em nenhum momento da reunião, e nenhum desenvolvedor futuro vai questionar por que o módulo de webhooks usa o mesmo padrão de controller/service/repository dos outros módulos.

A decisão arquitetural real que havia sido tomada na reunião era outra: **manter o módulo de webhooks dentro do monólito existente em vez de criar um microsserviço separado**. Essa sim tem alternativa real, trade-off explícito e será questionada quando alguém sugerir extração no futuro. A ADR foi reescrita com esse foco.

### Iteração 3 — RFC atualizada após geração dos ADRs

A RFC foi gerada antes dos ADRs. Quando os ADRs foram criados, a ADR-007 (snapshot do payload) surgiu de uma decisão tomada após a saída da maioria dos participantes da reunião e não estava referenciada na RFC. O link foi adicionado manualmente após a geração dos ADRs. Isso reforça a recomendação do desafio de gerar os ADRs antes da RFC.

---

## Como navegar a entrega

### Estrutura de arquivos

```
docs/
├── PRD.md              ← comece aqui: contexto de produto, por que e o quê
├── RFC.md              ← proposta técnica e alternativas descartadas
├── FDD.md              ← especificação de implementação com contratos e fluxos
├── TRACKER.md          ← rastreabilidade de cada item à transcrição ou ao código
└── adrs/
    ├── ADR-001-outbox-no-mysql.md
    ├── ADR-002-worker-em-processo-separado.md
    ├── ADR-003-retry-backoff-e-dlq.md
    ├── ADR-004-hmac-sha256-secret-por-endpoint.md
    ├── ADR-005-at-least-once-com-event-id.md
    ├── ADR-006-webhook-no-monolito-existente.md
    └── ADR-007-snapshot-payload-na-insercao.md

processo-executado-exercicio/
├── PROCESSO.md             ← este arquivo
├── resumo-reuniao.md       ← resumo organizado da transcrição
├── comparacoes-prd/
│   ├── PRD_ChatGPT.md
│   └── PRD_ClaudeAI.md
└── prompts/
    └── prompt-resumo-reuniao.md
```

### Ordem sugerida de leitura

1. `TRANSCRICAO.md` — entenda o que foi discutido na reunião original
2. `processo-executado-exercicio/resumo-reuniao.md` — versão organizada do que foi decidido e por quê
3. `docs/PRD.md` — visão de produto: problema, escopo e métricas
4. `docs/adrs/ADR-001` a `ADR-007` — as decisões técnicas individuais, cada uma com contexto e trade-off
5. `docs/RFC.md` — proposta técnica consolidada e questões em aberto
6. `docs/FDD.md` — especificação completa de implementação com contratos e fluxos
7. `docs/TRACKER.md` — para verificar a origem de qualquer item nos documentos

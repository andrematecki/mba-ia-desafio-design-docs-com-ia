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
├── PROCESSO.md             ← este arquivo (cópia em docs separados)
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

---

---

# Enunciado original do desafio

> O conteúdo abaixo é o enunciado original mantido para referência.

# Da Reunião ao Documento: Design Docs Gerados por IA

## Descrição

Neste desafio você vai transformar a transcrição de uma reunião técnica em um pacote completo de design docs, usando IA como ferramenta principal de produção.

**Cenário:** uma empresa que opera um Order Management System (OMS) em produção vai construir uma nova feature, um Sistema de Webhooks de Notificação de Pedidos. A decisão técnica já foi tomada em uma reunião entre tech lead, PM, engenheiros e segurança, mas nada foi registrado além da transcrição da call (`TRANSCRICAO.md`).

**Sua tarefa:** produzir, a partir da transcrição e do código existente, a documentação técnica da feature, em nível acionável o suficiente para o time de engenharia iniciar a implementação.

## Sobre o uso de IA

A IA é sua ferramenta principal de produção neste desafio. Você vai usá-la para ler o código, analisar a transcrição, estruturar os documentos e gerar o conteúdo final. O que se espera de você é o papel de maestro: definir o que precisa ser feito, formular bons prompts, revisar criticamente o que a IA entrega, corrigir e refinar até o resultado ficar consistente.

## Estrutura do desafio

O desafio consiste em produzir um **pacote de design docs**: PRD, RFC, FDD, ADRs, Tracker e o README do processo a partir da transcrição e do código.

## Objetivo

Entregar, em um repositório público no GitHub (fork do repositório base), o seguinte pacote de documentação:

- PRD (Product Requirement Document) da feature
- RFC (Request for Comments) com a proposta técnica da solução, submetida à equipe para revisão
- FDD (Feature Design Document) da feature
- Entre 5 e 8 ADRs (Architecture Decision Records) das decisões discutidas
- Tracker de rastreabilidade ligando cada item à origem na transcrição ou no código
- README atualizado documentando o processo de produção

Toda informação registrada nos documentos deve ser rastreável à transcrição ou ao código fonte da aplicação. Não é permitido inventar requisitos, decisões ou restrições sem origem identificável.

### O pacote de documentos e o papel de cada um

Os documentos não se repetem: cada um opera em uma **altura** diferente. Antes de produzir, entenda a fronteira entre eles: conteúdo duplicado entre documentos é sinal de que algo está no lugar errado.

| Documento | Papel | Altura | Pergunta que responde |
| --- | --- | --- | --- |
| **PRD** | Problema, público, escopo e métricas de sucesso | Produto / negócio | *Por que e o quê?* |
| **RFC** | Proposta técnica da solução para revisão: abordagem geral, alternativas e questões em aberto | Arquitetura | *Como pretendemos resolver, e o que ainda está em aberto?* |
| **ADRs** | Cada decisão arquitetural isolada, com contexto e consequências | Decisão pontual | *Por que decidimos exatamente assim?* |
| **FDD** | Especificação de implementação: fluxos, contratos, erros, integração com o código | Implementação | *Como construir, em detalhe?* |
| **Tracker** | Rastreabilidade de cada item ao código ou à transcrição | Transversal | *De onde veio cada coisa?* |

Em uma frase: o **RFC propõe e abre para revisão**, os **ADRs registram cada decisão fechada** e o **FDD detalha como construir**. O RFC é conciso (2 a 4 páginas) e fala em decisão; o FDD é profundo e fala em implementação. Não repita no RFC o nível de detalhe do FDD.

## Contexto

### A aplicação existente

O repositório base contém uma aplicação Node.js + TypeScript funcional: um Order Management System com módulos de autenticação, usuários, clientes, produtos e pedidos. Banco MySQL via Prisma. O ciclo de vida do pedido tem máquina de estados controlada, controle transacional de estoque e auditoria de mudanças de status.

A aplicação não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks. Esse vácuo é proposital. É exatamente o que a feature discutida na reunião pretende preencher.

Seus documentos vão precisar referenciar componentes do código existente, como a estrutura modular, a máquina de estados, a transação do `changeStatus`, as classes de erro, o padrão de códigos de erro, o middleware `requireRole`, o error middleware centralizado e o logger Pino. Use a IA para mapear esses pontos a partir do código.

### A transcrição

O arquivo `TRANSCRICAO.md` contém a gravação literal da reunião técnica. Cinco participantes discutem por aproximadamente 55 minutos no formato `[hh:mm] Nome: fala`.

A transcrição inclui decisões fechadas, requisitos funcionais explícitos, restrições, ganchos com o código existente, pontos descartados ou adiados para fases futuras e detalhes técnicos secundários. Nem tudo que foi mencionado vira requisito. Algumas coisas foram explicitamente descartadas, outras foram adiadas. Identificar o que NÃO entra é tão importante quanto identificar o que entra. Use a IA com prompts dirigidos para fazer essa filtragem, não pedidos genéricos.

## Tecnologias e ferramentas

Liberdade total na escolha de ferramentas de IA. Você pode usar qualquer combinação de Claude, ChatGPT, Cursor, Copilot Chat, Gemini, agentes, prompts customizados, skills ou plugins. Aproveite os prompts e plugins disponibilizados pelo professor durante o curso como ponto de partida.

Os documentos devem ser entregues em formato Markdown.

A entrega é puramente documental: você não deve mexer no código da aplicação (`src/`, `prisma/`, `tests/`, configurações). O código serve de contexto e referência.

## Requisitos

### 1. PRD da feature

Produza o arquivo `docs/PRD.md` cobrindo a feature de Sistema de Webhooks de Notificação de Pedidos. O PRD deve seguir o formato apresentado no curso e incluir, no mínimo, as seguintes seções:

- Resumo e contexto da feature
- Problema e motivação
- Público-alvo e cenários de uso
- Objetivos e métricas de sucesso
- Escopo (incluso e fora de escopo)
- Requisitos funcionais
- Requisitos não funcionais
- Decisões e trade-offs principais
- Dependências
- Riscos e mitigação
- Critérios de aceitação
- Estratégia de testes e validação

A seção "Fora de escopo" deve listar explicitamente pelo menos 2 itens descartados ou adiados durante a reunião.

### 2. RFC da feature

Produza o arquivo `docs/RFC.md` com a proposta técnica da solução, no formato de um documento submetido à equipe para revisão. O RFC opera em nível de arquitetura: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto. É um documento conciso (2 a 4 páginas); o detalhamento de implementação fica no FDD. Deve seguir o formato apresentado no curso e incluir, no mínimo:

- Metadados (autor, status, data, revisores); use os participantes da reunião como revisores
- Resumo executivo (TL;DR) da proposta
- Contexto e problema
- Proposta técnica (visão geral da solução, sem descer ao detalhe de implementação do FDD)
- Alternativas consideradas (pelo menos 2 alternativas reais discutidas e descartadas na reunião, cada uma com o trade-off que levou ao descarte)
- Questões em aberto (pelo menos 2 pontos levantados na reunião e não decididos ou adiados)
- Impacto e riscos
- Decisões relacionadas (links para os ADRs correspondentes)

O RFC não deve duplicar o detalhamento do FDD. Ele responde "o que propomos e por quê"; o "como construir" em detalhe fica no FDD.

### 3. FDD da feature

Produza o arquivo `docs/FDD.md` detalhando o "como implementar" da feature. O FDD é o documento mais técnico e precisa estar acionável o suficiente para um desenvolvedor pegar e começar a codar. Deve seguir o formato apresentado no curso e incluir, no mínimo:

- Contexto e motivação técnica
- Objetivos técnicos
- Escopo e exclusões
- Fluxos detalhados (criação do evento na outbox, processamento pelo worker, retry, DLQ)
- Contratos públicos (endpoints HTTP com payloads de exemplo, headers, status codes, semântica)
- Matriz de erros previstos com códigos no padrão `WEBHOOK_*`
- Estratégias de resiliência (timeouts, retries, backoff, fallback)
- Observabilidade (métricas, logs, tracing)
- Dependências e compatibilidade
- Critérios de aceite técnicos
- Riscos e mitigação

Seção obrigatória adicional, específica deste desafio: **"Integração com o sistema existente"**. Esta seção deve nomear pelo menos 4 caminhos de arquivo reais do código base e descrever como o módulo de webhooks vai se integrar com cada um (por exemplo, como o método `changeStatus` será estendido, como as classes de erro existentes serão reutilizadas).

### 4. ADRs

Produza entre 5 e 8 ADRs em arquivos separados dentro de `docs/adrs/`, nomeados no formato `ADR-NNN-titulo-em-kebab-case.md` (ex: `ADR-001-outbox-no-mysql.md`).

Cada ADR deve seguir o formato MADR (ou variante padrão) com no mínimo as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).

Pelo menos 1 ADR deve referenciar explicitamente arquivos, módulos ou padrões do código existente.

O conjunto de ADRs deve cobrir, no mínimo, 5 das 6 decisões principais discutidas na reunião:

- Padrão Outbox no MySQL
- Política de retry com backoff e DLQ
- Autenticação HMAC-SHA256 com secret por endpoint
- Garantia at-least-once com `X-Event-Id`
- Worker em processo separado em polling
- Reuso dos padrões existentes do projeto

Decisões técnicas secundárias (formato de payload, timeouts, headers, entre outras) podem virar ADRs adicionais ou ficar apenas no FDD, conforme você considerar mais adequado.

### 5. Tracker de Rastreabilidade

Produza o arquivo `docs/TRACKER.md`, uma tabela markdown que mapeia cada item registrado nos seus documentos à origem na transcrição ou no código. O tracker funciona como uma referência cruzada: permite que qualquer leitor entenda de onde veio cada decisão, requisito ou restrição, e garante que a documentação está alinhada com o que foi efetivamente discutido e com o que existe no código.

O tracker não é um conceito padrão do mercado nem é um documento abordado diretamente no curso. É uma exigência específica deste desafio que ajuda a manter a integridade da documentação contra alucinações da IA.

Formato obrigatório da tabela:

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |

Onde:

- **ID**: identificador único do item (ex: PRD-FR-01, RFC-ALT-02, FDD-CONTRATO-03, ADR-002)
- **Documento**: arquivo onde o item aparece (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-...md`)
- **Tipo**: Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, entre outros
- **Conteúdo (resumo)**: descrição de uma linha do item
- **Fonte**: `TRANSCRICAO` ou `CODIGO`
- **Localização**: para `TRANSCRICAO`, timestamp + nome do falante (ex: `[09:17] Diego`). Para `CODIGO`, caminho do arquivo (ex: `src/modules/orders/order.service.ts`).

Cobertura mínima: pelo menos 80% dos itens identificáveis nos seus documentos devem ter linha correspondente no tracker.

### 6. README com o processo

O `README.md` na raiz do repositório base contém este enunciado. Substitua o conteúdo dele pela documentação do seu processo de produção. Você pode manter um link ou seção fazendo referência ao enunciado original se quiser, mas o foco do novo conteúdo é descrever sua jornada.

Estrutura obrigatória do novo README:

- **Sobre o desafio**: 1 a 2 parágrafos descrevendo a tarefa em suas palavras
- **Ferramentas de IA utilizadas**: lista das ferramentas que você usou, com breve nota sobre o papel de cada uma
- **Workflow adotado**: como você organizou o trabalho. Em que ordem produziu os documentos, como organizou a interação com a IA
- **Prompts customizados**: pelo menos 2 prompts relevantes que você escreveu ou adaptou, mostrados em blocos de código
- **Iterações e ajustes**: descreva os principais momentos em que a IA gerou algo errado ou superficial e você teve que corrigir. Quantas iterações principais até chegar ao resultado final
- **Como navegar a entrega**: caminho dos arquivos entregues e ordem sugerida de leitura

---

## Critérios de Aceite

A entrega é avaliada contra os critérios abaixo. Todos são obrigatórios.

### PRD (`docs/PRD.md`)

- ☐ Arquivo existe e está em Markdown
- ☐ Contém todas as seções obrigatórias listadas no requisito 1
- ☐ Identifica no mínimo 8 requisitos funcionais discutidos na reunião
- ☐ Inclui pelo menos 1 objetivo com métrica e meta quantitativa
- ☐ Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião
- ☐ Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação

### RFC (`docs/RFC.md`)

- ☐ Arquivo existe e está em Markdown
- ☐ Contém todas as seções obrigatórias listadas no requisito 2
- ☐ Seção "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião, cada uma com o trade-off que motivou o descarte
- ☐ Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião
- ☐ Referencia, com link, pelo menos 2 ADRs do pacote

### FDD (`docs/FDD.md`)

- ☐ Arquivo existe e está em Markdown
- ☐ Contém todas as seções obrigatórias listadas no requisito 3
- ☐ Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes
- ☐ Matriz de erros usa códigos com prefixo `WEBHOOK_`
- ☐ Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base
- ☐ Seção "Observabilidade" cita métricas, logs e tracing

### ADRs (`docs/adrs/ADR-NNN-*.md`)

- ☐ Pasta `docs/adrs/` contém entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md`
- ☐ Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- ☐ O conjunto cobre pelo menos 5 das 6 decisões principais listadas no requisito 4
- ☐ Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base

### Tracker (`docs/TRACKER.md`)

- ☐ Arquivo existe e segue o formato de tabela definido no requisito 5
- ☐ Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
- ☐ Pelo menos 70% das linhas têm Fonte = `TRANSCRICAO` com timestamp válido no formato `[hh:mm] Nome`
- ☐ Pelo menos 5 linhas têm Fonte = `CODIGO` com caminho de arquivo real

### README (`README.md`)

- ☐ Contém todas as seções obrigatórias listadas no requisito 6
- ☐ Lista pelo menos 1 ferramenta de IA utilizada
- ☐ Mostra pelo menos 2 prompts customizados em blocos de código
- ☐ Descreve pelo menos 2 iterações ou ajustes concretos feitos durante a produção

### Consistência geral

- ☐ Nenhum requisito, decisão ou restrição registrada nos documentos contradiz a transcrição ou o código
- ☐ Nenhum arquivo de código mencionado nos documentos é inexistente no repositório

---

## Estrutura obrigatória do entregável

```
.
├── README.md                              (substituído pelo aluno)
├── TRANSCRICAO.md                         (não alterar)
├── docs/
│   ├── PRD.md                             (preenchido pelo aluno)
│   ├── RFC.md                             (preenchido pelo aluno)
│   ├── FDD.md                             (preenchido pelo aluno)
│   ├── TRACKER.md                         (preenchido pelo aluno)
│   └── adrs/
│       ├── ADR-001-titulo-curto.md
│       ├── ADR-002-titulo-curto.md
│       ├── ADR-003-titulo-curto.md
│       ├── ADR-004-titulo-curto.md
│       ├── ADR-005-titulo-curto.md
│       └── ... (até 8 ADRs)
├── src/                                   (não alterar)
├── prisma/                                (não alterar)
├── tests/                                 (não alterar)
└── ... (demais arquivos do boilerplate)
```

A entrega deve ser feita como repositório público no GitHub, a partir de fork do repositório base do desafio.

## Repositório base

O repositório base do desafio contém a aplicação completa, a transcrição e a estrutura de pastas pra você preencher:

https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia

## Ordem de execução sugerida

1. **Fork e setup**: faça o fork do repositório base e clone localmente.
2. **Contextualização com IA**: forneça à IA acesso ao código (via Claude Code, Cursor lendo o repo, ou colando trechos relevantes) e à transcrição. Peça uma exploração inicial para entender estrutura, padrões e o que a feature precisa endereçar.
3. **ADRs primeiro**: identifique e produza as decisões principais antes dos demais documentos. As decisões formam o esqueleto do "como implementar".
4. **RFC**: consolide a proposta técnica em cima das decisões. As alternativas descartadas e as questões em aberto da reunião têm lugar natural aqui. Referencie os ADRs já escritos.
5. **FDD**: com as decisões formalizadas e a proposta consolidada, o desenho técnico se constrói em cima delas. Lembre da seção obrigatória "Integração com o sistema existente".
6. **PRD**: produza o PRD por último entre os grandes documentos. Como ele é mais alto nível, com RFC, FDD e ADRs em mãos vira praticamente uma consolidação.
7. **Tracker**: monte em paralelo com os outros documentos ou no fim, varrendo os documentos prontos.
8. **README do processo**: deixe por último, quando o processo já está completo e você pode documentá-lo com clareza.
9. **Revisão final**: passe pela checklist de critérios de aceite item por item antes do push final.
10. **Itere**: é esperado que o processo demande 3 a 5 ciclos de geração, revisão crítica, ajuste de prompt e nova geração. Se você gerou tudo de primeira sem ajustes, os documentos provavelmente estão genéricos demais.

## Dicas Finais

A qualidade do prompt determina a qualidade do documento. Prompts vagos do tipo "gere um PRD a partir dessa transcrição" produzem documentos vazios e genéricos. Aproveite os prompts disponibilizados pelo professor no curso como base e adapte-os ao contexto deste desafio.

O tracker é seu melhor aliado contra alucinações da IA. Se você não consegue preencher a coluna "Localização" para uma linha do PRD ou do FDD, é sinal de que aquela informação não tem origem identificável e provavelmente foi inventada pela IA. Ajuste ou remova.

Cuidado com o que NÃO entra na documentação. A reunião descarta explicitamente algumas ideias. Se essas coisas aparecerem como requisito nos seus documentos, é sinal de que a IA não está sendo cuidadosa com o que você pediu.

A restrição de não alterar o código da aplicação é absoluta: o código serve de contexto e referência, e o entregável é puramente documental.

Itere bastante. Os primeiros documentos que a IA gerar provavelmente serão superficiais ou redundantes. Volte com correções, peça refinamento de pontos específicos, peça para remover trechos vagos, peça exemplos concretos. O resultado final deve parecer escrito por alguém que pensou no problema com a IA ao lado, não por alguém que copiou e colou da transcrição.

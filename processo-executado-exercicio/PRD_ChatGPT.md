### PRD: Order Management System (OMS) Sistema de Webhooks de Notificação de Pedidos

Versão: Não definido na reunião Data: Não definido na reunião
Responsável: Não definido na reunião

------------------------------------------------------------------------

### Resumo

Implementar um sistema de webhooks no Order Management System (OMS) para
permitir que clientes B2B sejam notificados automaticamente quando
houver alteração no status de seus pedidos.

A solução elimina a necessidade de polling contínuo ao endpoint
`GET /orders`, utilizando o padrão Outbox para garantir consistência
entre a atualização do pedido e a publicação do evento.

------------------------------------------------------------------------

### Contexto e problema

Público-alvo

-   Clientes B2B integrados ao OMS
-   Atlas Comercial
-   MaxDistribuição
-   Nova Cargo
-   Demais clientes

Cenários de uso chave

-   Receber notificações automáticas quando o status de um pedido mudar.
-   Eliminar a necessidade de polling.

Onde essa feature será implantada

-   Sistema existente: Order Management System (OMS)

Problemas priorizados

-   Polling contínuo aumenta custo de integração e tráfego.
-   Maior latência para descoberta de mudanças.
-   Risco de perda de clientes estratégicos.

------------------------------------------------------------------------

### Objetivos e métricas

  -----------------------------------------------------------------------
  Objetivo                Métrica                 Meta
  ----------------------- ----------------------- -----------------------
  Permitir notificações   Não definido na reunião Entrega em menos de 10
  automáticas                                     segundos

  Eliminar polling        Não definido na reunião Não definido na reunião
  -----------------------------------------------------------------------

------------------------------------------------------------------------

### Escopo

Incluso

-   CRUD de webhooks
-   Filtro de eventos
-   Outbox
-   Worker dedicado
-   Polling a cada 2 segundos
-   Retry com backoff exponencial (1m,5m,30m,2h,12h)
-   DLQ
-   Replay manual
-   Histórico de entregas
-   HMAC-SHA256
-   Rotação de secret com grace period de 24 horas
-   HTTPS obrigatório
-   Snapshot do payload
-   Garantia at-least-once
-   Auditoria do replay

Fora de escopo

-   Dashboard
-   Notificação por e-mail
-   Rate limiting
-   Garantia de ordenação global
-   Arquivamento automático da Outbox

------------------------------------------------------------------------

### Requisitos funcionais

Foram identificados os seguintes requisitos:

-   FR-001 Cadastro de webhook
-   FR-002 Atualização de webhook
-   FR-003 Exclusão de webhook
-   FR-004 Listagem de webhooks
-   FR-005 Inserção do evento na Outbox dentro da mesma transação do
    changeStatus
-   FR-006 Worker de processamento
-   FR-007 Retry automático
-   FR-008 Persistência em DLQ
-   FR-009 Replay manual da DLQ (ADMIN)
-   FR-010 Histórico de entregas
-   FR-011 Rotação de secret
-   FR-012 Filtragem de eventos na inserção da Outbox
-   FR-013 Auditoria do replay

------------------------------------------------------------------------

### Requisitos não funcionais

Performance

-   Polling de 2 segundos
-   Timeout HTTP de 10 segundos

Disponibilidade

-   Não definida na reunião

Segurança e autorização

-   HTTPS obrigatório
-   HMAC-SHA256
-   Secret por endpoint
-   Rotação de secret
-   Endpoint de replay exige ADMIN

Observabilidade

-   Histórico de entregas
-   Auditoria do replay

Confiabilidade e integridade de dados

-   Outbox na mesma transação do pedido
-   Snapshot do payload
-   Garantia at-least-once

Compatibilidade e portabilidade

-   JSON sobre HTTP

Compliance

-   Não definido na reunião

Acessibilidade no frontend consumidor

-   Não aplicável

------------------------------------------------------------------------

### Arquitetura e abordagem

Abordagem

-   Padrão Outbox em MySQL
-   Worker separado da API

Componentes

-   webhook_outbox
-   webhook_dead_letter
-   Worker
-   Módulo webhooks
-   Order Service

Integrações

-   Order Service
-   Endpoints HTTP dos clientes

### Decisões e trade-offs

#### Decisão: Utilizar Outbox

-   **Justificativa:** Garantir atomicidade.
-   **Trade-off:** Necessidade de worker dedicado.

#### Decisão: Garantia at-least-once

-   **Justificativa:** Simplicidade e padrão de mercado.
-   **Trade-off:** Cliente deve deduplicar usando X-Event-Id.

------------------------------------------------------------------------

### Dependências

#### Técnica: Integração com Order Service

Inserção da publicação na mesma transação do changeStatus.

#### Organizacional: Revisão de segurança

Revisão obrigatória antes do deploy.

------------------------------------------------------------------------

### Riscos e mitigação

#### Cliente indisponível

-   **Probabilidade:** média
-   **Impacto:** Eventos não entregues.
-   **Mitigação:**
    -   Retry exponencial.
    -   Persistência em DLQ.
-   **Plano de contingência:** Replay manual.

#### Crescimento da Outbox

-   **Probabilidade:** média
-   **Impacto:** Redução de desempenho.
-   **Mitigação:**
    -   Processamento em lotes.
    -   Índices por status e created_at.
-   **Plano de contingência:** Arquivamento futuro.

------------------------------------------------------------------------

### Critérios de aceitação

-   CRUD funcional.
-   Eventos registrados na mesma transação do pedido.
-   Worker envia notificações.
-   Retry implementado.
-   DLQ funcional.
-   Replay ADMIN funcional.
-   Assinatura HMAC válida.
-   Histórico disponível.

------------------------------------------------------------------------

### Testes e validação

Tipos de teste obrigatórios

-   Testes unitários.
-   Testes ponta a ponta.
-   Revisão de segurança.

Estratégia de validação

-   Revisão técnica.
-   Revisão de segurança.
-   Validação do fluxo completo de alteração de status até entrega do
    webhook.

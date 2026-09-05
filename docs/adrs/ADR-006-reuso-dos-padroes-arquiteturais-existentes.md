# ADR-006: Reuso dos Padrões Arquiteturais e Técnicos Existentes da Codebase

## Status
Aceito

## Contexto
O Order Management System (OMS) possui uma arquitetura modular consolidada em Node.js com TypeScript. A estrutura de código segue convenções bem delineadas que facilitam a manutenção, previsibilidade de testes e observabilidade:
1. Organização modular orientada a domínios em diretórios dedicados dentro de [`src/modules/`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/modules/) (ex: `orders/`, `customers/`, `products/`, `users/`, `auth/`), contendo camadas estritas:
   * `*.controller.ts`: manipulação de requisição e resposta HTTP;
   * `*.service.ts`: regras de negócio e orquestração de domínio;
   * `*.repository.ts`: operações de persistência via Prisma;
   * `*.routes.ts`: mapeamento de rotas e injeção de middlewares;
   * `*.schemas.ts`: contratos de entrada e validações estruturadas com Zod.
2. Tratamento centralizado de erros em [`src/middlewares/error.middleware.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/error.middleware.ts) baseado na classe base [`AppError`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/shared/errors/app-error.ts), suportando códigos canônicos e serialização homogênea.
3. Observabilidade com logging estruturado via Pino em [`src/shared/logger/index.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/shared/logger/index.ts).
4. Controle de acesso baseado em roles (`ADMIN`, `OPERATOR`) através de [`src/middlewares/auth.middleware.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/auth.middleware.ts).

Ao projetar o subsistema de webhooks, a equipe avaliou se criaria novos paradigmas arquiteturais ou bibliotecas de terceiros para orquestração de jobs, ou se manteria fidelidade aos padrões do projeto.

## Decisão
1. **Estrutura Modular Canônica em `src/modules/webhooks/`:**
   * Criar o novo domínio seguindo rigorosamente a topologia existente:
     * `webhook.controller.ts`
     * `webhook.service.ts`
     * `webhook.repository.ts`
     * `webhook.routes.ts`
     * `webhook.schemas.ts`
     * `webhook.processor.ts` (lógica de execução e despacho invocada pelo entry-point [`src/worker.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/worker.ts)).
2. **Integração Transacional Não Intrusiva em `OrderService`:**
   * No método `changeStatus` de [`src/modules/orders/order.service.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts), a publicação na outbox será feita através de uma função auxiliar `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client da transação (`tx: Prisma.TransactionClient`). Evita-se assim a injeção do repositório inteiro de webhooks na camada de pedidos.
3. **Padronização de Erros com Prefixo Obrigatório `WEBHOOK_*`:**
   * Todos os erros de domínio estenderão [`AppError`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/shared/errors/app-error.ts), definindo códigos de erro expressivos no padrão do projeto:
     * `WEBHOOK_NOT_FOUND`
     * `WEBHOOK_INVALID_URL`
     * `WEBHOOK_SECRET_REQUIRED`
     * `WEBHOOK_DELIVERY_FAILED`
     * `WEBHOOK_DEAD_LETTER_NOT_FOUND`
   * O middleware existente [`src/middlewares/error.middleware.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/error.middleware.ts) interceptará e formatará esses erros automaticamente sem requerer modificações de infraestrutura.
4. **Reuso de Logger e Autenticação:**
   * O worker e a API utilizarão a instância existente do logger Pino (`logger` em [`src/shared/logger/index.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/shared/logger/index.ts)).
   * As rotas de CRUD utilizarão o middleware `authenticate`, e as rotas críticas (como o replay da DLQ) utilizarão `requireRole('ADMIN')` de [`src/middlewares/auth.middleware.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/auth.middleware.ts).

## Alternativas Consideradas
* **Adoção de Frameworks Pesados de Jobs (ex: BullMQ / Agenda):** Descartado porque demandaria introduzir Redis ou MongoDB na stack, quebrando a consistência do ecossistema centrado em MySQL + Prisma e impondo dependências externas adicionais.
* **Criação de Microserviço Independente para Webhooks:** Descartado nesta fase porque aumentaria a complexidade de deploy, governança de repositórios e observabilidade para um time enxuto, sendo muito mais eficiente manter o módulo coeso no monorepo existente.

## Consequências
### Positivas
* **Curva de Aprendizado Zero:** Qualquer desenvolvedor familiarizado com os módulos `orders` ou `customers` compreenderá imediatamente a implementação de `webhooks`.
* **Zero Alterações no Pipeline de Erros e Logs:** O ecossistema existente absorve a nova feature sem necessidade de refatorações estruturais.
* **Manutenibilidade e Testabilidade:** Estrutura clara para testes unitários com mocks de repository e testes de integração com banco de dados de teste.

### Negativas e Trade-offs
* **Acoplamento ao Prisma ORM:** O modelo de persistência continua acoplado ao Prisma Client gerado e ao ciclo de vida de transações interativas (`$transaction`).

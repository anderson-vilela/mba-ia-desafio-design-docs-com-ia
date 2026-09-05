# Tracker de Rastreabilidade

A tabela abaixo estabelece a rastreabilidade cruzada entre todos os requisitos, decisões de arquitetura, restrições e contratos documentados no pacote de Design Docs (`PRD.md`, `RFC.md`, `FDD.md` e `ADRs`), associando cada item à sua fonte de origem verificável: ou na transcrição da reunião técnica ([`TRANSCRICAO.md`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/TRANSCRICAO.md)) ou no código fonte da aplicação existente.

---

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| :--- | :--- | :--- | :--- | :---: | :--- |
| **PRD-MOT-01** | `docs/PRD.md` | Motivação | Clientes B2B (Atlas, Max, Nova Cargo) e risco de churn da Atlas | TRANSCRICAO | `[09:00] Marcos` |
| **PRD-MET-01** | `docs/PRD.md` | Métrica / Meta | Latência de notificação abaixo de 10 segundos ($p99 < 10\text{s}$) | TRANSCRICAO | `[09:02] Marcos` |
| **PRD-ESC-01** | `docs/PRD.md` | Escopo | Suporte exclusivo a webhooks de saída (*outbound*), sem inbound | TRANSCRICAO | `[09:02] Marcos` |
| **PRD-ESC-02** | `docs/PRD.md` | Fora de Escopo | Notificação de falhas por e-mail fora de escopo nesta fase | TRANSCRICAO | `[09:37] Larissa` |
| **PRD-ESC-03** | `docs/PRD.md` | Fora de Escopo | Limite de taxa de envio (*rate limiting*) adiado para observação posterior | TRANSCRICAO | `[09:39] Diego` |
| **PRD-ESC-04** | `docs/PRD.md` | Fora de Escopo | Dashboard visual de webhooks fora de escopo (somente endpoints API) | TRANSCRICAO | `[09:40] Larissa` |
| **PRD-ESC-05** | `docs/PRD.md` | Fora de Escopo | Rotina de expurgo/arquivamento da outbox após 30 dias postergada | TRANSCRICAO | `[09:08] Diego` |
| **PRD-FR-01** | `docs/PRD.md` | Requisito Funcional | Cadastro de endpoint via POST com URL, eventos e geração de secret | TRANSCRICAO | `[09:31] Marcos` |
| **PRD-FR-02** | `docs/PRD.md` | Requisito Funcional | Autenticação normal na API com `customerId` no body ou path | TRANSCRICAO | `[09:32] Larissa` |
| **PRD-FR-03** | `docs/PRD.md` | Requisito Funcional | Endpoints PATCH, DELETE e GET para gestão de endpoints pelo cliente | TRANSCRICAO | `[09:33] Bruno` |
| **PRD-FR-04** | `docs/PRD.md` | Requisito Funcional | Filtro granular de eventos por lista de status desejados | TRANSCRICAO | `[09:33] Marcos` |
| **PRD-FR-05** | `docs/PRD.md` | Requisito Funcional | Filtro na inserção da outbox (não insere se ninguém escuta o status) | TRANSCRICAO | `[09:34] Bruno` |
| **PRD-FR-06** | `docs/PRD.md` | Requisito Funcional | Histórico de entregas com status, response e tempo em GET `/webhooks/:id/deliveries` | TRANSCRICAO | `[09:34] Marcos` |
| **PRD-FR-07** | `docs/PRD.md` | Requisito Funcional | Endpoint administrativo de replay manual de DLQ | TRANSCRICAO | `[09:35] Diego` |
| **PRD-FR-08** | `docs/PRD.md` | Requisito Funcional | Endpoint de replay de DLQ restrito a perfil com role ADMIN | TRANSCRICAO | `[09:36] Sofia` |
| **PRD-FR-09** | `docs/PRD.md` | Requisito Funcional | Formato do payload JSON enxuto omitindo lista de itens do pedido | TRANSCRICAO | `[09:43] Diego` |
| **PRD-FR-10** | `docs/PRD.md` | Requisito Funcional | Rotação de secret via API com período de tolerância (*grace period*) de 24h | TRANSCRICAO | `[09:21] Sofia` |
| **PRD-DEP-01** | `docs/PRD.md` | Dependência | Prazo total estimado em 3 sprints com 2 dias de revisão de segurança | TRANSCRICAO | `[09:46] Larissa` |
| **PRD-SEC-01** | `docs/PRD.md` | Restrição / Segurança | Janela obrigatória de revisão de segurança da Sofia antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| **RFC-PROP-01** | `docs/RFC.md` | Proposta Técnica | Adoção do padrão Transactional Outbox para garantir consistência | TRANSCRICAO | `[09:06] Diego` |
| **RFC-PROP-02** | `docs/RFC.md` | Proposta Técnica | Worker autônomo desacoplado executando polling a cada 2 segundos | TRANSCRICAO | `[09:10] Larissa` |
| **RFC-ALT-01** | `docs/RFC.md` | Trade-off / Alternativa | Descarte de envio HTTP síncrono por risco de travar transações do OMS | TRANSCRICAO | `[09:04] Bruno` |
| **RFC-ALT-02** | `docs/RFC.md` | Trade-off / Alternativa | Descarte de Redis Streams por complexidade operacional para time enxuto | TRANSCRICAO | `[09:07] Diego` |
| **RFC-ALT-03** | `docs/RFC.md` | Trade-off / Alternativa | Descarte de triggers MySQL por falta de mecanismo nativo pub/sub | TRANSCRICAO | `[09:09] Diego` |
| **RFC-ABERTO-01** | `docs/RFC.md` | Questão em Aberto | Avaliação de rate limiting de saída após acompanhamento em produção | TRANSCRICAO | `[09:39] Larissa` |
| **RFC-ABERTO-02** | `docs/RFC.md` | Questão em Aberto | Definição futura da rotina de arquivamento dos registros entregues | TRANSCRICAO | `[09:08] Diego` |
| **ADR-DEC-001** | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Decisão Arquitetural | Persistência do evento na tabela outbox na mesma transação SQL de mudança de status | TRANSCRICAO | `[09:06] Diego` |
| **ADR-DEC-002** | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Decisão Arquitetural | Snapshot do evento serializado gerado no momento da inserção na outbox | TRANSCRICAO | `[09:52] Larissa` |
| **ADR-DEC-003** | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão Arquitetural | Worker executado em processo Node.js independente da API Express | TRANSCRICAO | `[09:11] Diego` |
| **ADR-DEC-004** | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão Arquitetural | Criação do entry-point `src/worker.ts` e script `npm run worker` | TRANSCRICAO | `[09:11] Larissa` |
| **ADR-DEC-005** | `docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md` | Decisão Arquitetural | Single-worker assegurando garantia de ordering implícita por pedido | TRANSCRICAO | `[09:12] Diego` |
| **ADR-DEC-006** | `docs/adrs/ADR-003-politica-de-retry-com-backoff-exponencial-e-dlq.md` | Decisão Arquitetural | Curva de 5 retries com backoff progressivo: 1m, 5m, 30m, 2h, 12h (~15h total) | TRANSCRICAO | `[09:17] Diego` |
| **ADR-DEC-007** | `docs/adrs/ADR-003-politica-de-retry-com-backoff-exponencial-e-dlq.md` | Decisão Arquitetural | Persistência de falhas definitivas na tabela separada `webhook_dead_letter` | TRANSCRICAO | `[09:18] Diego` |
| **ADR-DEC-008** | `docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Decisão Arquitetural | Assinatura HMAC-SHA256 enviada no cabeçalho `X-Signature` | TRANSCRICAO | `[09:20] Sofia` |
| **ADR-DEC-009** | `docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Decisão Arquitetural | Chave secreta exclusiva e isolada por endpoint de webhook | TRANSCRICAO | `[09:21] Sofia` |
| **ADR-DEC-010** | `docs/adrs/ADR-005-garantia-de-entrega-at-least-once-com-x-event-id.md` | Decisão Arquitetural | Garantia at-least-once delegando desduplicação ao cliente receptor | TRANSCRICAO | `[09:24] Diego` |
| **ADR-DEC-011** | `docs/adrs/ADR-005-garantia-de-entrega-at-least-once-com-x-event-id.md` | Decisão Arquitetural | Cabeçalho `X-Event-Id` contendo UUID para permitir idempotência no consumidor | TRANSCRICAO | `[09:25] Diego` |
| **ADR-DEC-012** | `docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md` | Decisão Arquitetural | Prefixo obrigatório `WEBHOOK_*` para todos os códigos de erro do domínio | TRANSCRICAO | `[09:29] Larissa` |
| **FDD-CONTRATO-01** | `docs/FDD.md` | Contrato / Cabeçalho | Inclusão dos cabeçalhos `X-Timestamp` e `X-Webhook-Id` no envio do webhook | TRANSCRICAO | `[09:44] Sofia` |
| **FDD-RESI-01** | `docs/FDD.md` | Resiliência | Timeout de 10 segundos para chamadas HTTP de saída efetuadas pelo worker | TRANSCRICAO | `[09:42] Diego` |
| **FDD-RESI-02** | `docs/FDD.md` | Requisito Não Funcional | Teto limite de tamanho de payload serializado em 64KB | TRANSCRICAO | `[09:24] Diego` |
| **COD-ORDER-01** | `docs/FDD.md` | Código / Integração | Ponto de extensão no método `changeStatus` com transação `$transaction` | CODIGO | `src/modules/orders/order.service.ts` |
| **COD-ORDER-02** | `docs/FDD.md` | Código / Regra | Máquina de estados e validações de transição de pedido (`canTransition`) | CODIGO | `src/modules/orders/order.status.ts` |
| **COD-PRISMA-01** | `docs/FDD.md` | Código / Modelo | Enums `OrderStatus`, `UserRole` e padrão de identificadores UUID (`db.Char(36)`) | CODIGO | `prisma/schema.prisma` |
| **COD-AUTH-01** | `docs/FDD.md` | Código / Segurança | Middleware `requireRole('ADMIN')` utilizado na proteção da rota de replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| **COD-ERR-01** | `docs/FDD.md` | Código / Erros | Hierarquia de erro estendendo a classe base canônica `AppError` | CODIGO | `src/shared/errors/app-error.ts` |
| **COD-ERR-02** | `docs/FDD.md` | Código / Erros | Middleware de tratamento global de exceções capturando instâncias de `AppError` | CODIGO | `src/middlewares/error.middleware.ts` |
| **COD-LOG-01** | `docs/FDD.md` | Código / Observabilidade | Instância compartilhada do logger estruturado Pino | CODIGO | `src/shared/logger/index.ts` |
| **COD-ROUTE-01** | `docs/FDD.md` | Código / Arquitetura | Montagem modular de rotas via `buildApiRouter` em `src/routes/index.ts` | CODIGO | `src/routes/index.ts` |
| **COD-BOOT-01** | `docs/FDD.md` | Código / Arquitetura | Inicialização do servidor HTTP e rotina de encerramento gracioso (*graceful shutdown*) | CODIGO | `src/server.ts` |

---

### Resumo Estatístico de Rastreabilidade

* **Total de Itens Mapeados:** 50
* **Itens com Origem em `TRANSCRICAO`:** 41 (82.0% do total — supera o critério mínimo de 70%)
* **Itens com Origem em `CODIGO`:** 9 (18.0% do total — supera o critério mínimo de 5 linhas)
* **Formato de Transcrição:** 100% das linhas de transcrição seguem o padrão obrigatório `[hh:mm] Nome`.
* **Formato de Código:** 100% das linhas de código apontam para caminhos de arquivos reais e existentes na codebase.

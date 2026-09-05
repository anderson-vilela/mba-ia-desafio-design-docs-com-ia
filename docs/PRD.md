# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

## 1. Resumo e Contexto da Feature
A plataforma Order Management System (OMS) gerencia o ciclo completo de compras e movimentações logísticas para diversos clientes comerciais. Três clientes corporativos B2B estratégicos — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — solicitaram formalmente a capacidade de receber notificações automáticas em tempo real sobre cada mudança de status de seus pedidos (`PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`).

O objetivo desta iniciativa é introduzir o **Sistema de Webhooks de Notificação de Pedidos**, permitindo que sistemas externos recebam eventos de saída (*outbound webhooks*) de forma assíncrona, segura, resiliente e rastreável, eliminando consultas manuais e gargalos de integração.

---

## 2. Problema e Motivação
Hoje, a plataforma não oferece nenhum mecanismo reativo de notificação de eventos. Como consequência:
1. **Ineficiência Operacional:** Os clientes B2B executam rotinas frequentes de sondagem (*polling*) repetitivas no endpoint `GET /orders`, gerando alto consumo inútil de conexões no banco de dados e tráfego desnecessário na API.
2. **Latência na Cadeia de Suprimentos:** A sincronização das operações de faturamento, transporte e entrega fica atrasada pelo intervalo entre as consultas dos parceiros.
3. **Ameaça Crítica de Churn:** A **Atlas Comercial** comunicou formalmente que migrará suas operações para um concorrente direto caso a integração em tempo real não esteja disponível até o término do trimestre vigente (fim de novembro).

A implementação do sistema de webhooks resolve definitivamente essas dores, estancando o risco comercial e modernizando a camada de integração do OMS.

---

## 3. Público-Alvo e Cenários de Uso

### 3.1. Público-Alvo
* **Engenheiros e Integradores de Clientes B2B:** Desenvolvedores das empresas parceiras (Atlas, MaxDistribuição, Nova Cargo) responsáveis por configurar seus servidores para receber notificações e integrar com seus respectivos ERPs e WMSs.
* **Operadores e Administradores do OMS:** Equipes internas de atendimento, operações e suporte técnico da nossa plataforma responsáveis por auditar entregas e intervir em casos de falhas.

### 3.2. Cenários de Uso
* **Cenário 1 — Notificação de Pagamento e Início de Separação:** Um pedido tem o status alterado para `PAID`. O sistema do cliente B2B recebe a notificação em segundos, acionando automaticamente a ordem de separação no centro de distribuição próprio.
* **Cenário 2 — Sincronização Logística de Despacho:** O operador do OMS atualiza o pedido para `SHIPPED`. O webhook é disparado com a notificação, permitindo que a transportadora do cliente gere a rota de entrega imediatamente.
* **Cenário 3 — Rotação Preventiva de Chaves:** O time de segurança do parceiro corporativo rotaciona a chave secreta via API, usufruindo de 24 horas de tolerância (*grace period*) para atualizar seus servidores sem indisponibilidade.
* **Cenário 4 — Recuperação Operacional de Falha:** Um servidor de cliente sofre manutenção de 2 horas. O OMS realiza tentativas com backoff exponencial sem perda de eventos; se esgotar as tentativas, o operador aciona o replay manual via API com privilégios de administrador.

---

## 4. Objetivos e Métricas de Sucesso

| Categoria | Métrica | Meta Quantitativa | Forma de Medição |
| :--- | :--- | :---: | :--- |
| **Performance** | Latência de Notificação ($p99$) | $< 10\text{ segundos}$ | Tempo decorrido entre o commit transacional de `changeStatus` e o primeiro despacho HTTP efetuado pelo worker. |
| **Eficiência** | Redução de Polling em `GET /orders` | $\ge 75\%$ | Comparação do volume de requisições GET originadas dos parceiros B2B 30 dias antes e após o lançamento. |
| **Negócio** | Retenção de Clientes B2B | $100\%$ | Renovação contratual e permanência integral da Atlas Comercial, MaxDistribuição e Nova Cargo. |
| **Confiabilidade** | Taxa de Sucesso de Entregas | $\ge 99.5\%$ | Percentual de eventos entregues com sucesso (considerando as 5 tentativas de retry) em relação ao total gerado. |
| **Cronograma** | Prazo de Entrega | $\le 3\text{ sprints}$ | Homologação e deploy em produção até o fim de novembro de 2026, com validação de segurança aprovada. |

---

## 5. Escopo

### 5.1. No Escopo
* Cadastro, listagem, atualização e remoção de endpoints de webhook via API autenticada.
* Configuração granular de filtro de eventos de interesse por endpoint (ex: escutar apenas `PAID` e `SHIPPED`).
* Geração automatizada de secret única por endpoint e suporte à rotação suave com janela de 24 horas.
* Enfileiramento transacional atômico (*Outbox Pattern*) no MySQL junto à mudança de status do pedido.
* Processo de worker autônomo em Node.js com ciclo de polling de 2 segundos.
* Assinatura criptográfica obrigatória de payload com `HMAC-SHA256` no cabeçalho `X-Signature`.
* Validação estrita de protocolo HTTPS para todos os endpoints receptores.
* Política de 5 retries com backoff exponencial ($1\text{m}, 5\text{m}, 30\text{m}, 2\text{h}, 12\text{h}$) e timeout de 10s.
* Tabela de Dead Letter Queue (DLQ) para falhas permanentes com endpoint de replay manual restrito a `ADMIN`.
* Histórico de entregas dos últimos disparos para auditoria do cliente (`GET /webhooks/:id/deliveries`).

### 5.2. Fora de Escopo (Explicitamente Descartados ou Adiados)
* **Webhooks Inbound (Recepção de Terceiros):** Apenas webhooks de saída (*outbound*) serão implementados. O OMS não receberá chamadas de webhook disparadas por sistemas externos. *(Descartado na reunião)*
* **Alertas e Notificações de Falha por E-mail:** O envio de e-mails para operadores do cliente após falhas consecutivas de entrega está fora do escopo desta fase, sendo postergado para ciclos futuros de produto. *(Adiado na reunião)*
* **Interface Gráfica (Dashboard Web):** Toda a gestão será realizada exclusivamente via endpoints da API REST; a implementação de UI no frontend foi delegada para roadmap futuro. *(Descartado na reunião)*
* **Expurgo Automático de Eventos Antigos da Outbox:** Rotinas automáticas de limpeza de registros da outbox após 30 dias foram adiadas para uma sprint posterior de sustentação técnica. *(Adiado na reunião)*
* **Rate Limiting / Throttling de Saída:** Mecanismos adaptativos de controle de taxa de saída por parceiro serão apenas monitorados em produção nesta fase inicial. *(Adiado na reunião)*

---

## 6. Requisitos Funcionais

* **RF-01 (Cadastro de Endpoints):** O sistema deve fornecer endpoint `POST /webhooks/endpoints` permitindo que clientes autenticados cadastrem URLs receptoras (obrigatoriamente HTTPS), descrição e os status de pedido que desejam monitorar.
* **RF-02 (Geração de Chave Secreta Exclusiva):** O sistema deve gerar automaticamente uma secret criptográfica segura e individual para cada endpoint criado, retornando-a na resposta do cadastro.
* **RF-03 (Filtro Granular de Eventos):** O sistema deve permitir que o cliente filtre quais transições de status (`OrderStatus`) deseja receber. Caso uma alteração de pedido não corresponda a nenhum endpoint inscrito, nenhum evento deve ser gravado na outbox.
* **RF-04 (Gestão de Endpoints):** O sistema deve fornecer endpoints para listar (`GET /webhooks/endpoints`), atualizar (`PATCH /webhooks/endpoints/:id`) e desativar/excluir (`DELETE /webhooks/endpoints/:id`) os endpoints vinculados a um cliente.
* **RF-05 (Rotação Suave de Chave Secreta):** O sistema deve permitir a rotação de secret via `POST /webhooks/endpoints/:id/rotate-secret`, mantendo a secret anterior funcional por exatamente 24 horas (*grace period*) para transição segura sem indisponibilidade.
* **RF-06 (Snapshot Transacional de Evento):** Ao executar `changeStatus`, o sistema deve gravar atomicamente na tabela `webhook_outbox` o snapshot serializado do evento em JSON contendo `orderId`, `orderNumber`, `fromStatus`, `toStatus` e `totalCents`, omitindo listas de itens detalhadas para manter a mensagem enxuta.
* **RF-07 (Assinatura Criptográfica HMAC-SHA256):** O despachador de eventos deve assinar digitalmente o corpo raw da requisição utilizando a secret exclusiva do endpoint e transmiti-lo no cabeçalho `X-Signature`.
* **RF-08 (Cabeçalhos de Rastreabilidade e Idempotência):** Toda requisição HTTP enviada para o parceiro deve conter os cabeçalhos `X-Event-Id` (UUID único), `X-Timestamp` (timestamp de emissão) e `X-Webhook-Id` (identificador do endpoint).
* **RF-09 (Resiliência com Backoff Exponencial):** Caso a entrega falhe por timeout (>10s), erro de rede ou retorno HTTP não-$2xx$, o sistema deve retentar a entrega em até 5 tentativas espaçadas nos intervalos de 1min, 5min, 30min, 2h e 12h.
* **RF-10 (Isolamento em Dead Letter Queue):** Após esgotadas as 5 tentativas de retentativa, o evento deve ter seu status alterado para `FAILED` e seus dados persistidos na tabela `webhook_dead_letter`.
* **RF-11 (Replay Administrativo com Auditoria):** O sistema deve fornecer o endpoint `POST /admin/webhooks/dead-letter/:id/replay`, acessível exclusivamente por usuários com role `ADMIN`, para reprocessar mensagens da DLQ, registrando em log quem ordenou a operação.
* **RF-12 (Consulta de Histórico de Entregas):** O sistema deve fornecer o endpoint `GET /webhooks/endpoints/:id/deliveries` para que os integradores consultem as últimas tentativas de envio, status de sucesso/falha, código HTTP de retorno e tempo de resposta.

---

## 7. Requisitos Não Funcionais

* **RNF-01 (Latência de Despacho):** O tempo entre o commit da alteração de status do pedido e o início do disparo HTTP pelo worker deve ser inferior a 10 segundos no percentil 99 ($p99 < 10\text{s}$).
* **RNF-02 (Confiabilidade de Entrega):** Semântica *at-least-once*; nenhum evento confirmado na base relacional pode ser descartado silenciosamente.
* **RNF-03 (Tamanho Limite de Mensagem):** O payload de notificação serializado não deve ultrapassar o limite estrito de 64KB, prevenindo exaustão de buffer e ataques de DoS.
* **RNF-04 (Timeout de Rede Externa):** Toda chamada de saída efetuada pelo worker deve possuir timeout rígido de 10 segundos configurado via `AbortSignal`.
* **RNF-05 (Segurança em Trânsito e Validação):** Apenas URLs sob o esquema criptografado `https://` são aceitas. Esquemas inseguros (`http://`) e endereços de rede privada/loopback devem ser bloqueados para prevenção contra Server-Side Request Forgery (SSRF).
* **RNF-06 (Desacoplamento de Processos):** O worker de disparo deve ser executado em um processo Node.js dedicado (`src/worker.ts`), sem compartilhar o ciclo de vida ou o event loop do servidor HTTP da API (`src/server.ts`).
* **RNF-07 (Padronização de Erros):** Todas as falhas geradas pelo módulo devem utilizar a estrutura canônica da classe `AppError` e códigos no padrão prefixado `WEBHOOK_*`.
* **RNF-08 (Observabilidade Estruturada):** Todos os eventos de envio e processamento devem ser registrados em JSON estruturado através do Pino logger, incluindo `eventId`, `webhookId`, `customerId`, `statusCode` e `durationMs`.

---

## 8. Decisões e Trade-offs Principais

* **Transactional Outbox no MySQL vs. Broker Externo (Redis Streams / RabbitMQ):** Priorizou-se a simplicidade arquitetural e a facilidade de sustentação pelo time enxuto, eliminando o problema de *dual-write* e mantendo zero custos adicionais de infraestrutura.
* **Worker em Polling de 2s vs. Triggers Relacionais:** O MySQL não suporta pub/sub nativo (`LISTEN/NOTIFY`). O polling de 2 segundos garante robustez operacional e cumpre com folga a meta de latência de 10s.
* **HMAC com Secret por Endpoint vs. Secret Global da Plataforma:** O isolamento de chave por endpoint restringe o raio de explosão (*blast radius*) caso um cliente parceiro tenha suas credenciais comprometidas em logs.
* **Entrega At-Least-Once com Idempotência via `X-Event-Id` vs. Exactly-Once via 2PC:** Implementar *exactly-once* na internet pública com clientes heterogêneos impõe complexidade proibitiva. O envio do UUID no cabeçalho permite que os parceiros implementem desduplicação simples.
* **Reuso da Arquitetura Existente:** O subsistema de webhooks preserva a estrutura em camadas (`controller`, `service`, `repository`, `schemas`), reutiliza middlewares de autorização (`requireRole`) e herda a tipagem de erros (`AppError`).

---

## 9. Dependências
* **Internas:** Gancho de publicação inserido na transação do método `changeStatus` em [`src/modules/orders/order.service.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts).
* **Banco de Dados:** MySQL 8.0 gerenciado via migrações declarativas do Prisma ORM ([`prisma/schema.prisma`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/prisma/schema.prisma)).
* **Segurança e Revisão:** Janela mandatória de 2 dias úteis reservada para a Engenheira de Segurança Sofia revisar o algoritmo HMAC, a geração de segredos e os schemas Zod antes do deploy.
* **Sistemas Externos:** Disponibilidade e capacidade de atendimento dos servidores receptores dos clientes corporativos B2B na internet pública.

---

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Estratégia de Mitigação |
| :--- | :---: | :---: | :--- |
| **Instabilidade ou Manutenção Prolongada de Clientes B2B** | Média | Alto | Janela de retry resiliente com 5 tentativas espaçadas ao longo de quase 15 horas ($1\text{m}, 5\text{m}, 30\text{m}, 2\text{h}, 12\text{h}$) e isolamento em tabela de DLQ para não degradar a fila de outros parceiros. |
| **Vulnerabilidade a Ataques de SSRF (Server-Side Request Forgery)** | Média | Crítico | Validação estrita via Zod bloqueando URLs sem HTTPS, hosts de loopback (`localhost`, `127.0.0.1`) e faixas de IP privadas (RFC 1918), complementada por auditoria de segurança da Sofia. |
| **Sobrecarga de Conexões no Banco MySQL por Polling do Worker** | Baixa | Médio | Inicialização de pool de conexões Prisma exclusivo e enxuto para o processo do worker, associado a índices compostos otimizados em `(status, next_retry_at, created_at)`. |
| **Clientes Processando Mensagens em Duplicidade** | Alta | Médio | Emissão obrigatória do cabeçalho de desduplicação `X-Event-Id` acompanhado de documentação de integração orientando a implementação de tabela de idempotência no receptor. |

---

## 11. Critérios de Aceitação
* [ ] Endpoint `POST /webhooks/endpoints` cadastra novos destinos, valida HTTPS e gera secret exclusiva.
* [ ] Endpoint `POST /webhooks/endpoints/:id/rotate-secret` emite nova chave e tolera a chave antiga por 24 horas.
* [ ] Transição de status do pedido e inclusão do evento na tabela `webhook_outbox` são atômicas (reversão mútua em caso de falha).
* [ ] O processo `src/worker.ts` processa eventos a cada 2s e despacha requisições HTTP com timeout de 10s.
* [ ] Cabeçalho `X-Signature` confere perfeitamente com o cálculo `HMAC-SHA256(rawPayload, secret)`.
* [ ] Eventos que falham 5 vezes consecutivas são gravados em `webhook_dead_letter` com status `FAILED`.
* [ ] O endpoint `POST /admin/webhooks/dead-letter/:id/replay` é bloqueado para operadores comuns (`403 Forbidden`) e funcional para a role `ADMIN`.
* [ ] Latência ponta a ponta é medida e validada abaixo de 10 segundos no percentil 99 ($p99 < 10\text{s}$).

---

## 12. Estratégia de Testes e Validação
1. **Testes Unitários (Vitest):**
   * Validação dos schemas Zod (rejeição de URLs `http://`, IPs inválidos e eventos inexistentes).
   * Cálculo determinístico de assinatura HMAC-SHA256 e validação com chaves ativas e rotacionadas.
   * Lógica do algoritmo de cálculo de backoff exponencial e incremento de tentativas.
2. **Testes de Integração:**
   * Verificação de atomicidade em banco de testes MySQL: rollback forçado na transação de pedido comprova que nenhum evento órfão permanece na outbox.
   * Teste do isolamento de transações entre a API REST e o pool do worker.
3. **Testes End-to-End (E2E) com Mock Servers:**
   * Utilização de servidor HTTP simulado para validar:
     * Recebimento de headers obrigatórios (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`);
     * Comportamento diante de respostas `200 OK`, timeouts de 10 segundos e respostas `500 Internal Server Error`;
     * Transbordo com sucesso para a tabela `webhook_dead_letter` após 5 falhas consecutivas;
     * Replay administrativo reinserindo o evento na outbox como `PENDING`.
4. **Homologação e Validação de Segurança:**
   * Execução da auditoria formal de 2 dias úteis conduzida pela Engenheira de Segurança Sofia antes da liberação em produção.

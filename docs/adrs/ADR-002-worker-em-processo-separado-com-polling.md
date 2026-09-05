# ADR-002: Worker em Processo Separado com Polling de 2 Segundos

## Status
Aceito

## Contexto
Com a decisão de utilizar o padrão Outbox no MySQL ([ADR-001](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/docs/adrs/ADR-001-padrao-outbox-no-mysql.md)), é necessário definir o mecanismo pelo qual os eventos pendentes na tabela `webhook_outbox` serão lidos, despachados via HTTP e marcados como processados.

Os clientes B2B estabeleceram como requisito de negócio receber as notificações em um intervalo inferior a 10 segundos após a mudança de status do pedido. Além disso, a aplicação possui um ponto de entrada HTTP principal em [`src/server.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/server.ts) que atende às requisições da API REST.

Precisamos definir:
1. A arquitetura de execução do componente leitor (se embutido na API ou isolado);
2. A estratégia de detecção de novos eventos no banco de dados MySQL;
3. O modelo de concorrência e ordenação de entrega inicial.

## Decisão
1. **Processo Node.js Independente:** Criar um novo ponto de entrada dedicado [`src/worker.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/worker.ts), executado via script `npm run worker`, completamente desacoplado do processo da API HTTP ([`src/server.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/server.ts)).
2. **Instância Dedicada do Prisma Client:** O processo do worker instanciará seu próprio `PrismaClient`, compartilhando as mesmas credenciais e variáveis de ambiente (`DATABASE_URL`), porém isolando o pool de conexões com o banco da API.
3. **Mecanismo de Polling a Cada 2 Segundos:** O worker executará um loop contínuo de leitura com intervalo de 2 segundos, buscando lotes de eventos com status `PENDING` ordenados por `created_at ASC`.
4. **Single-Worker e Ordenação Implícita:** Nesta primeira fase, a execução será configurada como uma instância única de worker (*single-worker*). Isso assegura a ordenação cronológica das notificações por pedido (`order_id`) sem exigir mecanismos complexos de locks distribuídos ou particionamento.

## Alternativas Consideradas
* **Triggers de Banco de Dados com Notificação Externa:** Descartado porque o MySQL não possui suporte nativo a eventos pub/sub ou comandos do tipo `LISTEN/NOTIFY` (existentes no PostgreSQL). Usar triggers exigiria invocações artificiais (como escrita em arquivos locais ou chamadas de UDFs C++), gerando fragilidade e complexidade operacional.
* **Worker Embutido no mesmo Processo da API Express (`src/server.ts`):** Descartado para proteger a estabilidade de ambos os serviços. Se a API sofrer reinício por deploy, indisponibilidade de rota ou esgotamento de event loop, o processamento de webhooks seria interrompido; do mesmo modo, chamadas HTTP massivas de webhooks poderiam degradar a latência das requisições web dos usuários.
* **Múltiplos Workers Concorrentes em Paralelo:** Descartado para a fase 1, pois múltiplos workers lendo simultaneamente a tabela outbox sem particionamento por hash de cliente ou lock pessimista poderiam causar entregas fora de ordem para o mesmo pedido (ex: `PROCESSING` chegar antes de `PAID`).

## Consequências
### Positivas
* **Isolamento de Ciclo de Vida e Recursos:** Falhas, crashes de memória ou rotinas pesadas de I/O do worker não afetam o throughput da API de pedidos e vice-versa.
* **Atendimento do SLA com Margem Segura:** O ciclo de polling de 2 segundos cumpre com folga a meta de entrega abaixo de 10 segundos exigida pelos parceiros B2B.
* **Simplicidade de Implementação:** Não requer infraestrutura adicional além do Node.js e MySQL já presentes no ecossistema da aplicação.

### Negativas e Trade-offs
* **Latência Mínima Inerente:** Qualquer evento terá um atraso de até 2 segundos antes do início do processamento.
* **Custo de Consultas Frequentes (Empty Polling):** Quando não há eventos pendentes, o worker continua executando consultas `SELECT` periódicas no MySQL.
* **Gargalo de Escalabilidade Vertical:** Manter *single-worker* limita o throughput máximo de entrega; expansões futuras exigirão particionamento por `customer_id` ou sharding de filas.

# ADR-001: Padrão Outbox no MySQL para Notificação de Eventos

## Status
Aceito

## Contexto
A plataforma opera um Order Management System (OMS) com persistência em MySQL via Prisma ORM. Três grandes clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) solicitaram notificações em tempo real sempre que o status de seus pedidos for alterado, evitando polling excessivo no endpoint `GET /orders`.

A transação de mudança de status de pedido em [`src/modules/orders/order.service.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts) (`changeStatus`) já executa operações críticas e concorrentes:
1. Atualização da tabela `orders`;
2. Inserção de registro na tabela `order_status_history`;
3. Atualização transacional de estoque de produtos (`debitStock` ou `replenishStock` na tabela `products`).

Disparar chamadas HTTP síncronas para os endpoints dos clientes dentro dessa transação colocaria o sistema em risco de indisponibilidade em cascata, travamentos por lentidão de rede e inconsistências graves (por exemplo, necessidade de rollback na alteração do pedido caso a chamada HTTP falhe, ou o status mudar mas a notificação externa ser perdida se a aplicação reiniciar).

## Decisão
Adotar o **Transactional Outbox Pattern** utilizando o próprio banco relacional **MySQL** já existente:
1. Uma nova tabela denominada `webhook_outbox` será criada no banco de dados via Prisma ([`prisma/schema.prisma`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/prisma/schema.prisma)).
2. A inserção do evento na tabela `webhook_outbox` ocorrerá dentro da **mesma transação SQL** gerenciada pelo Prisma Client (`prisma.$transaction`) no método `changeStatus` de [`src/modules/orders/order.service.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/modules/orders/order.service.ts).
3. Caso a transação do pedido falhe ou faça rollback, o evento do webhook será cancelado atomicamente junto com as alterações do pedido.
4. O payload do evento será gravado já serializado em JSON (snapshot do momento da alteração), prevenindo inconsistências temporais se o pedido sofrer alterações posteriores antes do envio.
5. Os identificadores da tabela utilizarão UUID (`db.Char(36)`), alinhando-se à modelagem existente do projeto.
6. A tabela conterá índices compostos nos campos `status` (pendente, processando, falhou, entregue) e `created_at` para otimizar a leitura do worker.

## Alternativas Consideradas
* **Disparo HTTP Síncrono no `OrderService`:** Descartado porque acopla a latência e estabilidade de sistemas externos à transação principal de negócio, podendo travar a mudança de status de outros pedidos e inviabilizar a transação em caso de indisponibilidade temporária do cliente B2B.
* **Fila de Mensageria / Redis Streams / RabbitMQ:** Descartado pela equipe devido ao princípio da simplicidade e tamanho reduzido do time de engenharia. Introduzir uma nova tecnologia de infraestrutura adicionaria complexidade operacional desnecessária (manutenção de cluster Redis/RabbitMQ, sincronização dual-write e monitoramento de novos serviços), enquanto o MySQL existente supre com folga a volumetria atual.

## Consequências
### Positivas
* **Atomicidade garantida (Dual-Write eliminado):** Não há risco de disparar evento de pedido cujo commit falhou, nem de confirmar alteração de pedido sem registrar o respectivo webhook.
* **Isolamento de falhas:** A indisponibilidade de rede ou falha no servidor de um cliente B2B não impacta as operações de pedido da nossa plataforma.
* **Sem novas dependências de infraestrutura:** Utiliza a infraestrutura de MySQL e o pool de conexões do Prisma já configurados.

### Negativas e Trade-offs
* **Sobrecarga de I/O no banco relacional:** Criação e atualização de registros de eventos concorrem com as operações transacionais do OMS.
* **Necessidade de limpeza futura (Housekeeping):** Registros processados e entregues demandarão política futura de arquivamento/expurgo (fora do escopo inicial) para não inflar a tabela indefinidamente.

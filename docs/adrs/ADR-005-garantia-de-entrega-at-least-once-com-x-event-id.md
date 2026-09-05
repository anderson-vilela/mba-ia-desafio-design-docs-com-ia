# ADR-005: Garantia de Entrega At-Least-Once com Desduplicação via X-Event-Id

## Status
Aceito

## Contexto
Em sistemas distribuídos assíncronos e redes não confiáveis, falhas de conectividade (por exemplo, queda na conexão após o servidor receptor processar o evento, mas antes do recebimento do status HTTP `200 OK` pelo remetente) causam retransmissões inevitáveis.

Para a feature de webhooks do OMS, precisamos escolher qual semântica de entrega será garantida pela plataforma:
1. *At-most-once* (no máximo uma vez, onde eventos podem se perder);
2. *At-least-once* (ao menos uma vez, onde eventos nunca são perdidos, mas podem ser duplicados);
3. *Exactly-once* (exatamente uma vez, garantindo entrega única sem repetições).

## Decisão
1. **Semântica de Entrega At-Least-Once:**
   * A plataforma garantirá que todo evento de mudança de status persistido na `webhook_outbox` será entregue com sucesso pelo menos uma vez ou encaminhado para a DLQ após o esgotamento dos retries.
   * Não haverá perda silenciosa de mensagens.
2. **Geração de UUID Único por Evento:**
   * No momento em que a transação de pedido insere o registro na tabela `webhook_outbox`, um identificador UUIDv4 imutável (`event_id`) será gerado.
3. **Cabeçalho de Idempotência `X-Event-Id`:**
   * O worker enviará obrigatoriamente esse UUID no cabeçalho HTTP `X-Event-Id` e dentro do payload JSON (`event_id`).
4. **Desduplicação e Idempotência Delegadas ao Consumidor:**
   * A responsabilidade de desduplicação é transferida para o cliente receptor, que deverá registrar os `event_id` já processados em sua base de dados e ignorar ou responder idempotentemente (`200 OK`) a requisições com IDs repetidos.
   * Essa diretriz será documentada explicitamente nos contratos de integração da plataforma.

## Alternativas Consideradas
* **Entrega Exactly-Once Baseada em Coordenação Bidirecional:** Descartada por exigir protocolos pesados de commit distribuído em duas fases (2-Phase Commit / 2PC) ou mecanismos síncronos de checagem prévia de estado entre a plataforma e os servidores dos parceiros. Na internet pública com clientes heterogêneos, essa abordagem introduz alta fragilidade operacional e latências inaceitáveis.
* **Garantia At-Most-Once (Fire-and-Forget):** Descartada porque não atende aos requisitos de negócio dos clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo), que necessitam de consistência estrita nas atualizações de status de seus pedidos logísticos e financeiros.

## Consequências
### Positivas
* **Confiabilidade de Entrega Sem Perdas:** Nenhuma notificação de mudança de status de pedido é descartada por problemas transitórios de rede.
* **Alinhamento com Padrões de Mercado:** Segue as melhores práticas consolidadas de APIs públicas globais como Stripe, GitHub e Shopify.
* **Simplicidade e Resiliência da Arquitetura:** O despachador não precisa manter estados compartilhados de confirmação complexos antes de efetuar retransmissões seguras.

### Negativas e Trade-offs
* **Necessidade de Idempotência no Consumidor:** O parceiro B2B que consome o webhook deve obrigatoriamente implementar tabelas ou caches de idempotência (persistindo `event_id` processados) para evitar processar o mesmo evento em duplicidade.

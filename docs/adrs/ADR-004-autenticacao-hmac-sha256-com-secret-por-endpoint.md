# ADR-004: Autenticação HMAC-SHA256 com Secret por Endpoint e Rotação Suave

## Status
Aceito

## Contexto
O sistema enviará notificações com informações sensíveis de pedidos (identificadores, valores e clientes) para servidores externos na internet pública mantidos pelos clientes B2B.

Nesse cenário de saída (*outbound webhooks*), é imperativo que os sistemas receptores consigam:
1. Validar a autenticidade do remetente (garantir que a mensagem foi de fato emitida pela nossa plataforma);
2. Validar a integridade da mensagem (garantir que o payload JSON não foi interceptado ou modificado em trânsito);
3. Detectar e mitigar ataques de repetição (*replay attacks*);
4. Conter o raio de alcance (*blast radius*) caso as credenciais de um cliente sejam expostas.

## Decisão
1. **Assinatura Criptográfica HMAC-SHA256:**
   * O worker assinará o corpo da requisição HTTP (raw JSON payload) utilizando o algoritmo `HMAC-SHA256` provido pelo módulo nativo `crypto` do Node.js.
   * O hash resultante será transmitido no cabeçalho HTTP `X-Signature`.
2. **Secret Exclusiva por Endpoint:**
   * Cada registro de endpoint de webhook possuirá sua própria chave secreta criptográfica gerada no momento do cadastro.
   * Não haverá segredo global ou compartilhado entre múltiplos endpoints ou múltiplos clientes.
3. **Mecanismo de Rotação Suave com Grace Period de 24 Horas:**
   * A API disponibilizará um endpoint para rotação da secret (`POST /webhooks/endpoints/:id/rotate-secret`).
   * Durante as **24 horas subsequentes à rotação**, o worker continuará gerando e validando a assinatura de modo que ambas as credenciais sejam toleradas durante o período de transição nos sistemas do cliente, garantindo que a troca não cause interrupção de serviço.
4. **Proteção Contra Replay Attack:**
   * O cabeçalho `X-Timestamp` (em formato ISO 8601 ou UNIX epoch) será enviado junto com a requisição, permitindo que o cliente rejeite eventos com carimbo de tempo antigo.
5. **Cabeçalho `X-Webhook-Id`:**
   * O identificador do cadastro do webhook será enviado no cabeçalho `X-Webhook-Id` para simplificar o roteamento interno de clientes que cadastram múltiplos endpoints.
6. **HTTPS Obrigatório:**
   * Validação rigorosa no schema Zod ([`src/modules/customers/customer.schemas.ts`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/modules/customers/customer.schemas.ts) / `webhook.schemas.ts`): endpoints cadastrados com protocolo `http://` serão sumariamente rejeitados.

## Alternativas Consideradas
* **Secret Global da Plataforma:** Descartado veementemente pela equipe de segurança. O vazamento acidental da secret por um único cliente comprometeria a segurança e a integridade das mensagens de todos os demais integradores do ecossistema.
* **Mutual TLS (mTLS):** Descartado por impor sobrecarga de gestão de infraestrutura, distribuição e renovação periódica de certificados X.509 tanto para o time de infraestrutura interno quanto para os times de TI dos parceiros B2B.
* **API Key Estática em Header Customizado:** Descartado por não proteger a integridade do payload (não previne adulteração em trânsito) e por ser vulnerável a vazamento em logs de proxies reversos e gateways.

## Consequências
### Positivas
* **Conformidade com Padrões da Indústria:** Modelo idêntico ao adotado por referências de mercado como Stripe, GitHub e Shopify.
* **Isolamento de Credenciais:** Vazamento do segredo de uma integração afeta exclusivamente aquele endpoint específico.
* **Zero Downtime em Rotação:** O período de tolerância de 24 horas viabiliza a substituição controlada de chaves nos ambientes dos clientes sem perdas.

### Negativas e Trade-offs
* **Sobrecarga de Implementação para o Consumidor:** Os clientes B2B precisam codificar a rotina de validação do HMAC-SHA256 em suas aplicações receptoras.
* **Complexidade na Gestão de Estados de Chaves:** O modelo de dados deve armazenar tanto a chave ativa quanto a chave anterior junto ao timestamp de expiração do grace period.

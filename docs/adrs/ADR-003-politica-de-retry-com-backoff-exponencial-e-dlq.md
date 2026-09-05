# ADR-003: Política de Retry com Backoff Exponencial e Dead Letter Queue (DLQ)

## Status
Aceito

## Contexto
Endpoints de clientes B2B estão sujeitos a instabilidades transitórias de rede, falhas internas em seus servidores ou paradas programadas de manutenção que podem durar até duas horas. Se o worker falhar ao entregar uma notificação na primeira tentativa, o sistema deve adotar uma estratégia de resiliência sem travar a fila de eventos e sem retentar infinitamente.

Precisamos definir:
1. O número máximo de tentativas de reenvio;
2. Os intervalos da curva de retry (backoff exponencial);
3. O destino dos eventos que esgotarem as tentativas;
4. O mecanismo de reprocessamento e controle de acesso a eventos falhos.

## Decisão
1. **Política de 5 Tentativas com Backoff Exponencial:**
   * O sistema executará até **5 tentativas** de envio antes de considerar a falha permanente.
   * Os intervalos de espera entre as tentativas seguirão a progressão:
     * 1ª retentativa: **1 minuto**
     * 2ª retentativa: **5 minutos**
     * 3ª retentativa: **30 minutos**
     * 4ª retentativa: **2 horas**
     * 5ª retentativa: **12 horas**
   * A janela total de tolerância cobre aproximadamente 14 horas e 36 minutos a partir da primeira falha, garantindo tempo hábil para recuperação de manutenções prolongadas dos parceiros.
2. **Timeout HTTP de 10 Segundos:** Toda chamada HTTP realizada pelo worker terá um timeout estrito de 10 segundos. Caso o servidor de destino não responda nesse prazo, a tentativa será computada como falha por timeout.
3. **Dead Letter Queue (DLQ) em Tabela Separada:**
   * Eventos que falharem após a 5ª tentativa serão movidos para a tabela `webhook_dead_letter` no banco MySQL.
   * A tabela armazenará: identificador original, payload completo, motivo da última falha (código de erro e mensagem HTTP/rede), contagem de tentativas e timestamps.
4. **Endpoint Administrativo de Replay:**
   * Disponibilizar o endpoint `POST /admin/webhooks/dead-letter/:id/replay` para reinserir o evento na `webhook_outbox` com status `PENDING`.
   * **Controle de Acesso:** O endpoint exigirá perfil de acesso com a role `ADMIN` reutilizando o middleware [`requireRole('ADMIN')`](file:///home/anderson/projects/fullcycle/mba-ia-desafio-design-docs-com-ia/src/middlewares/auth.middleware.ts), além de registrar em log estruturado (Pino) o identificador do operador responsável pelo reprocessamento.

## Alternativas Consideradas
* **Retry Indefinido:** Descartado porque manteria eventos falhos congestionando indefinidamente a tabela outbox no caso de clientes desativados ou URLs permanentemente inacessíveis.
* **Retry Agressivo de 3 Tentativas (Janela de 30 minutos):** Descartado porque não acomodaria janelas conhecidas de manutenção dos clientes B2B (que frequentemente duram 2 horas ou mais).
* **Manter Falhas Permanentes na Própria Tabela Outbox (com flag `status: 'FAILED'`):** Descartado para manter a tabela operacional limpa e de alta performance para o worker, isolando a análise forense e reprocessamento manual em uma estrutura analítica dedicada (`webhook_dead_letter`).

## Consequências
### Positivas
* **Resiliência a Falhas Transitórias e Manutenções:** Cobertura de quase 15 horas protege contra indisponibilidades reais sem perda de dados.
* **Isolamento e Rastreabilidade de Erros:** A tabela de DLQ mantém um histórico claro das causas raízes de falha para auditoria técnica.
* **Governança Segura:** Apenas administradores do OMS podem acionar o reenvio de mensagens a partir da DLQ.

### Negativas e Trade-offs
* **Necessidade de Modelagem de Tabela Adicional:** Requer uma tabela extra no banco MySQL via Prisma Schema e endpoints adicionais na API.
* **Reprocessamento Manual:** A recuperação de mensagens que atingiram a DLQ requer intervenção manual via API administrativa.

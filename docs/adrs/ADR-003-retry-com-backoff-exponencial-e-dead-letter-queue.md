# ADR-003: Retry com backoff exponencial e Dead Letter Queue

## Status

Aceito

## Contexto

Com o worker definido (ADR-002), era preciso decidir o que fazer quando a entrega de um webhook falha — por exemplo, quando o endpoint do cliente está offline. Diego propôs backoff exponencial: tentar novamente após intervalos crescentes e, após um teto de tentativas, considerar falha permanente e mover o evento para uma fila de mortos (DLQ) (`[09:15] Diego`).

Houve debate sobre o número de tentativas. Bruno sugeriu 3, por ser mais agressivo (`[09:16] Bruno`). Diego discordou, citando um caso real: já houve cliente com indisponibilidade de duas horas por manutenção planejada, e três tentativas em 30 minutos matariam o evento antes dessa janela terminar (`[09:16] Diego`). Diego também descartou retry indefinido, por deixar eventos pendurados para sempre se o cliente sumir definitivamente (`[09:15] Diego`). O time fechou em 5 tentativas, com progressão de backoff de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas — uma janela total de quase 15 horas entre a primeira falha e a última tentativa (`[09:17] Diego`). Marcos considerou a janela aceitável: um cliente indisponível por 15 horas já teria um problema sério do lado dele (`[09:17] Marcos`).

Para o destino final das falhas permanentes, Diego propôs uma tabela separada `webhook_dead_letter`, com payload, motivo da falha e timestamp, em vez de apenas marcar como "failed" na própria tabela de outbox — isso mantém a leitura da outbox principal mais limpa e serve como evidência para debug e reprocessamento (`[09:18] Diego`). O reprocessamento é manual, via um endpoint administrativo `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente (`[09:18] Diego`).

Esse endpoint de replay foi definido como exigindo a role `ADMIN` — mexer na fila de entrega de notificação não é considerado tarefa de operador — e deve registrar em log quem executou o replay, para fins de auditoria (`[09:35]-[09:36] Sofia/Larissa`). O projeto já possui o middleware `requireRole(...roles)` em `src/middlewares/auth.middleware.ts`, com suporte a RBAC sobre os papéis `ADMIN`/`OPERATOR` — confirmado no código — que pode ser reutilizado diretamente para essa checagem.

## Decisão

Ao falhar o envio de um webhook, o worker reagenda a tentativa com backoff exponencial: 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, totalizando 5 tentativas. Esgotadas as tentativas, o evento é movido para uma tabela separada `webhook_dead_letter` (payload, motivo da falha, timestamp), removendo-o da outbox ativa. O reprocessamento de itens em DLQ é manual, via endpoint `POST /admin/webhooks/dead-letter/:id/replay`, protegido por `requireRole('ADMIN')` e com log de auditoria do usuário que executou o replay.

## Alternativas Consideradas

- **3 tentativas.** Rejeitada: agressiva demais, mataria o evento antes de cobrir janelas de indisponibilidade reais já observadas em clientes (ex.: 2h de manutenção planejada) (`[09:16] Diego`).
- **Retry indefinido com backoff.** Rejeitada: risco de eventos ficarem pendurados indefinidamente caso o cliente nunca volte a responder (`[09:15] Diego`).
- **Marcar como "failed" na própria tabela de outbox, sem tabela de DLQ separada.** Rejeitada: polui a leitura da outbox principal e dificulta manter evidência estruturada para debug/reprocessamento (`[09:18] Diego`).

## Consequências

**Positivas:**
- Janela de retry de ~15 horas cobre incidentes reais de indisponibilidade de clientes já observados, sem manter eventos pendurados para sempre (`[09:16]-[09:17] Diego`).
- DLQ em tabela própria mantém a outbox principal enxuta e fornece evidência auditável (payload, motivo, timestamp) para investigação e reprocessamento manual (`[09:18] Diego`).
- Reaproveita o mecanismo de RBAC já existente (`requireRole`, `src/middlewares/auth.middleware.ts`) para restringir o replay a administradores.

**Negativas:**
- Após esgotadas as 5 tentativas e passadas ~15 horas, a recuperação do evento deixa de ser automática e depende de intervenção manual via endpoint administrativo.
- Introduz uma tabela e um endpoint administrativo adicionais a serem construídos, testados e mantidos, incluindo o registro de auditoria do replay.

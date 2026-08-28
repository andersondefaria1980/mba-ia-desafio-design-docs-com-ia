# Tracker de Rastreabilidade

Este documento mapeia cada item identificável nas ADRs, no RFC, no FDD e no PRD à sua origem: um timestamp em `TRANSCRICAO.md` (`[hh:mm] Nome`) ou um caminho real de arquivo no código-fonte (`CODIGO`). Nenhum item aqui listado é inventado — se uma linha não tivesse uma localização real, ela teria sido removida do documento correspondente em vez de registrada aqui.

**Nota de metodologia:** listas que apenas repetem, com outras palavras, um item já registrado em outra seção do pacote (ex.: a lista "Escopo incluso" do PRD, que restate os próprios Requisitos Funcionais; a "Proposta técnica" do RFC, que resume as 7 ADRs; a lista "Fora de escopo" do FDD, idêntica à do PRD) não geram uma linha adicional no tracker — a rastreabilidade dessas afirmações já está coberta pela linha original do item que elas restatement. Isso evita inflar a tabela com duplicatas sem valor informacional novo.

## Resumo de cobertura

- **Total de linhas:** 157
- **Fonte = TRANSCRICAO:** 143 linhas (~91%) — muito acima do mínimo de 70% exigido
- **Fonte = CODIGO:** 14 linhas, todas com caminho de arquivo real — muito acima do mínimo de 5 exigido
- **Documentos cobertos:** 7 ADRs, RFC, FDD, PRD

## ADRs (`docs/adrs/`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL: evento de webhook inserido na mesma transação da mudança de status | TRANSCRICAO | [09:06] Diego |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker de webhooks roda em processo Node separado, com polling de 2 segundos | TRANSCRICAO | [09:09] Diego |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md | Decisão | Retry com 5 tentativas e backoff 1m/5m/30m/2h/12h, seguido de DLQ em tabela separada | TRANSCRICAO | [09:17] Diego |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Autenticação de entrega via HMAC-SHA256 com secret exclusiva por endpoint e rotação com grace period | TRANSCRICAO | [09:20] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Garantia de entrega at-least-once com deduplicação via X-Event-Id | TRANSCRICAO | [09:24] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md | Decisão | Módulo de webhooks reaproveita padrões existentes do OMS (módulos, AppError, logger, middlewares) | TRANSCRICAO | [09:30] Larissa |
| ADR-007 | docs/adrs/ADR-007-ordenacao-por-order-id-sem-garantia-global.md | Decisão | Ordenação garantida apenas por order_id via worker único, sem garantia global | TRANSCRICAO | [09:13] Diego |

## RFC (`docs/RFC.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CTX-01 | docs/RFC.md | Restrição | Clientes B2B pedem notificação em tempo real (<10s), sob risco de churn | TRANSCRICAO | [09:00] Marcos |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Disparo síncrono na transação de changeStatus descartado por travar mudança de status | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Fila externa (Redis Streams) descartada por exigir infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Garantia exactly-once descartada por exigir coordenação bilateral complexa | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Secret HMAC global descartada por ampliar o raio de impacto de um vazamento | TRANSCRICAO | [09:21] Sofia |
| RFC-ABERTO-01 | docs/RFC.md | Restrição | Rate limiting de envio ao cliente registrado como ponto em aberto | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| RFC-ABERTO-02 | docs/RFC.md | Restrição | Notificação de falha ao cliente (e-mail) adiada para fase futura | TRANSCRICAO | [09:37]-[09:38] Marcos/Larissa |
| RFC-ABERTO-03 | docs/RFC.md | Restrição | Escalonamento do worker para múltiplos processos classificado como problema futuro | TRANSCRICAO | [09:13] Diego |
| RFC-IMPACTO-01 | docs/RFC.md | Risco | Impacto de negócio: retenção de 3 clientes B2B ligada ao prazo de fim de novembro | TRANSCRICAO | [09:00], [09:45] Marcos |
| RFC-IMPACTO-02 | docs/RFC.md | Decisão | Impacto de engenharia: novo módulo + processo worker + 2 tabelas novas, estimativa de 3 sprints | TRANSCRICAO | [09:45]-[09:46] Larissa |
| RFC-IMPACTO-03 | docs/RFC.md | Restrição | Impacto de segurança: nova superfície de saída exige 2 dias de revisão dedicada da Sofia | TRANSCRICAO | [09:46] Sofia |
| RFC-RISCO-01 | docs/RFC.md | Risco | Endpoint do cliente indisponível por horas — mitigado por retry+DLQ | TRANSCRICAO | [09:15]-[09:18] Diego |
| RFC-RISCO-02 | docs/RFC.md | Risco | Vazamento de secret — mitigado por secret por endpoint + rotação | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| RFC-RISCO-03 | docs/RFC.md | Risco | Crescimento indefinido da outbox — mitigado por índices e arquivamento futuro | TRANSCRICAO | [09:08] Diego |
| RFC-RISCO-04 | docs/RFC.md | Risco | Worker único como gargalo de vazão — aceito como limitação conhecida | TRANSCRICAO | [09:12]-[09:13] Diego |

## FDD (`docs/FDD.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBJ-01 | docs/FDD.md | Requisito Não Funcional | Atomicidade entre mudança de status e registro do evento de webhook | TRANSCRICAO | [09:41] Diego |
| FDD-OBJ-02 | docs/FDD.md | Requisito Não Funcional | Latência efetiva <10s via polling de 2s | TRANSCRICAO | [09:09]-[09:10] Diego/Larissa |
| FDD-OBJ-03 | docs/FDD.md | Requisito Não Funcional | Sobreviver a indisponibilidade do cliente por até ~15h via retry | TRANSCRICAO | [09:15]-[09:17] Diego |
| FDD-OBJ-04 | docs/FDD.md | Requisito Não Funcional | Autenticidade/integridade via HMAC-SHA256 com blast radius limitado por endpoint | TRANSCRICAO | [09:20]-[09:21] Sofia |
| FDD-OBJ-05 | docs/FDD.md | Requisito Não Funcional | Semântica de entrega at-least-once com X-Event-Id para dedup | TRANSCRICAO | [09:24]-[09:25] Diego |
| FDD-OBJ-06 | docs/FDD.md | Restrição | Não introduzir infraestrutura nova, reaproveitando MySQL/Prisma/Pino/AppError | TRANSCRICAO | [09:07] Diego |
| FDD-DADOS-01 | docs/FDD.md | Decisão | Tabela webhook_endpoint: url/secret/customer_id/estado ativo/filtro de status/rotação | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Decisão | Tabela webhook_outbox: payload snapshot, event_id UUID, índice status/created_at | TRANSCRICAO | [09:06], [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Decisão | Tabela webhook_delivery: histórico de entregas (sucesso/falha, response, tempo) | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-04 | docs/FDD.md | Decisão | Tabela webhook_dead_letter separada para falhas permanentes | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-05 | docs/FDD.md | Restrição | Convenção de ID UUID e `@@map` snake_case seguindo padrão do schema.prisma | CODIGO | prisma/schema.prisma |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | Criação do evento: `publishWebhookEvent(tx,...)` chamado dentro da transação de `changeStatus` | TRANSCRICAO | [09:41]-[09:42] Bruno/Diego |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Processamento pelo worker: polling, montagem de headers, POST com timeout de 10s | TRANSCRICAO | [09:42] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Retry: recálculo de `next_attempt_at` conforme progressão de backoff | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Decisão | DLQ: move evento após 5ª falha, replay recoloca como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Requisito Funcional | Endpoint `POST /webhooks` — cadastro de webhook | TRANSCRICAO | [09:31]-[09:32] Marcos/Bruno |
| FDD-CONTRATO-02 | docs/FDD.md | Requisito Funcional | Endpoint `GET /webhooks` — listagem por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Requisito Funcional | Endpoint `PATCH /webhooks/:id` — edição | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Requisito Funcional | Endpoint `DELETE /webhooks/:id` — remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Requisito Funcional | Endpoint `GET /webhooks/:id/deliveries` — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | docs/FDD.md | Requisito Funcional | Endpoint `POST /webhooks/:id/secret/rotate` — rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | docs/FDD.md | Requisito Funcional | Endpoint `POST /admin/webhooks/dead-letter/:id/replay` — replay administrativo | TRANSCRICAO | [09:18], [09:35]-[09:36] Diego/Sofia/Larissa |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Código `WEBHOOK_NOT_FOUND` (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Restrição | Código `WEBHOOK_INVALID_URL` (422) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Restrição | Código `WEBHOOK_SECRET_REQUIRED` (400) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Restrição | Código `WEBHOOK_PAYLOAD_TOO_LARGE` (limite de 64KB) | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| FDD-ERRO-05 | docs/FDD.md | Restrição | Código `WEBHOOK_DELIVERY_TIMEOUT` (timeout de 10s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-06 | docs/FDD.md | Restrição | Código `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404) | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-01 | docs/FDD.md | Requisito Não Funcional | Timeout de 10s por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Requisito Não Funcional | Retry exponencial, 5 tentativas, ~15h de janela | TRANSCRICAO | [09:15]-[09:17] Diego |
| FDD-RESIL-03 | docs/FDD.md | Requisito Não Funcional | Fallback para DLQ sem bloquear demais eventos | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-04 | docs/FDD.md | Requisito Não Funcional | Isolamento de processo do worker frente a restarts da API | TRANSCRICAO | [09:11] Diego |
| FDD-RESIL-05 | docs/FDD.md | Requisito Não Funcional | Atomicidade transacional entre status e evento | TRANSCRICAO | [09:06], [09:41] Diego/Bruno |
| FDD-RESIL-06 | docs/FDD.md | Requisito Não Funcional | Grace period de 24h aceitando secret nova e antiga | TRANSCRICAO | [09:21] Sofia |
| FDD-OBS-01 | docs/FDD.md | Requisito Não Funcional | Métricas de fila pendente, taxa de sucesso/falha, tentativas até sucesso, DLQ | TRANSCRICAO | [09:08]-[09:10], [09:15]-[09:17] Diego |
| FDD-OBS-02 | docs/FDD.md | Restrição | Logs via Pino existente, com redact estendido para a secret do webhook | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Decisão | X-Event-Id como identificador de correlação de ponta a ponta (sem tracing dedicado) | TRANSCRICAO | [09:25] Diego |
| FDD-INTEG-01 | docs/FDD.md | Restrição | `changeStatus()` estendido para chamar `publishWebhookEvent` dentro da mesma transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Restrição | `OrderStatus`/`canTransition` reaproveitados para validar `status_filter` e transições | CODIGO | src/modules/orders/order.status.ts |
| FDD-INTEG-03 | docs/FDD.md | Restrição | Novos erros `WEBHOOK_*` estendem `AppError`/subclasses existentes | CODIGO | src/shared/errors/app-error.ts |
| FDD-INTEG-04 | docs/FDD.md | Restrição | Middleware de erro central absorve os novos erros sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Restrição | `authenticate`/`requireRole('ADMIN')` reaproveitados nos endpoints do módulo | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-06 | docs/FDD.md | Restrição | Logger Pino reaproveitado pela API e pelo worker | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-07 | docs/FDD.md | Restrição | Novo router de webhooks registrado ao lado dos routers existentes | CODIGO | src/routes/index.ts |
| FDD-INTEG-08 | docs/FDD.md | Restrição | `validate.middleware.ts` + novos schemas Zod seguindo padrão de `order.schemas.ts` | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INTEG-09 | docs/FDD.md | Restrição | Novas tabelas seguem convenções de `Order`/`OrderItem`/`OrderStatusHistory` | CODIGO | prisma/schema.prisma |
| FDD-DEP-01 | docs/FDD.md | Dependência | MySQL/Prisma já provisionados, sem infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| FDD-DEP-02 | docs/FDD.md | Dependência | HMAC via módulo `crypto` nativo do Node; nenhuma nova dependência externa listada | CODIGO | package.json |
| FDD-DEP-03 | docs/FDD.md | Dependência | Novo processo de runtime (worker) precisa ser refletido na topologia de deploy | TRANSCRICAO | [09:11] Diego |
| FDD-DEP-04 | docs/FDD.md | Restrição | Extensão de `changeStatus` é aditiva, sem quebrar contrato público existente | CODIGO | src/modules/orders/order.service.ts |
| FDD-CA-01 | docs/FDD.md | Critério de Aceitação | Mudança de status insere evento na outbox para cada endpoint interessado, na mesma transação | TRANSCRICAO | [09:34], [09:40]-[09:41] Diego/Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de Aceitação | Falha na inserção da outbox causa rollback da transação inteira | TRANSCRICAO | [09:40]-[09:41] Bruno/Diego |
| FDD-CA-03 | docs/FDD.md | Critério de Aceitação | Worker processa em ordem de `created_at`, ciclo de 2s | TRANSCRICAO | [09:08]-[09:09] Diego |
| FDD-CA-04 | docs/FDD.md | Critério de Aceitação | Falha reagenda tentativa conforme progressão 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-CA-05 | docs/FDD.md | Critério de Aceitação | Após 5 falhas, evento vai para `webhook_dead_letter` com motivo preenchido | TRANSCRICAO | [09:18] Diego |
| FDD-CA-06 | docs/FDD.md | Critério de Aceitação | Replay de DLQ restrito a `ADMIN`, com auditoria | TRANSCRICAO | [09:35]-[09:36] Sofia/Larissa |
| FDD-CA-07 | docs/FDD.md | Critério de Aceitação | Entrega inclui todos os headers padronizados e assinatura válida | TRANSCRICAO | [09:44]-[09:45] Diego/Sofia |
| FDD-CA-08 | docs/FDD.md | Critério de Aceitação | Grace period de 24h aceita ambas as secrets | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-09 | docs/FDD.md | Critério de Aceitação | Histórico de entregas retorna no máximo 100 registros | TRANSCRICAO | [09:34] Marcos |
| FDD-CA-10 | docs/FDD.md | Critério de Aceitação | Payload acima de 64KB não é enviado, falha registrada sem truncamento | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego/Larissa |
| FDD-RISCO-01 | docs/FDD.md | Risco | Cliente indisponível por horas — mitigado por retry+DLQ | TRANSCRICAO | [09:16]-[09:18] Diego |
| FDD-RISCO-02 | docs/FDD.md | Risco | Vazamento de secret — mitigado por secret por endpoint + rotação | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| FDD-RISCO-03 | docs/FDD.md | Risco | Outbox cresce indefinidamente — mitigado por índices e arquivamento futuro | TRANSCRICAO | [09:08] Diego |
| FDD-RISCO-04 | docs/FDD.md | Risco | Worker único como gargalo de vazão — limitação conhecida | TRANSCRICAO | [09:12]-[09:13] Diego |
| FDD-RISCO-05 | docs/FDD.md | Risco | Cliente recebe evento duplicado — mitigado por X-Event-Id | TRANSCRICAO | [09:24]-[09:26] Diego/Sofia/Marcos |
| FDD-RISCO-06 | docs/FDD.md | Risco | API reinicia e processamento para — mitigado por processo separado | TRANSCRICAO | [09:11] Diego |

## PRD (`docs/PRD.md`)

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-PUBLICO-01 | docs/PRD.md | Restrição | Público-alvo: sistemas de clientes B2B (Atlas, MaxDistribuição, Nova Cargo); consumo sempre outbound | TRANSCRICAO | [09:00], [09:02] Marcos/Sofia |
| PRD-CENARIO-01 | docs/PRD.md | Cenário de Uso | Cliente cadastra endpoint informando URL e status de interesse | TRANSCRICAO | [09:31]-[09:34] Marcos/Bruno |
| PRD-CENARIO-02 | docs/PRD.md | Cenário de Uso | Pedido muda de status e cliente recebe notificação HTTP assinada sem precisar consultar `GET /orders` | TRANSCRICAO | [09:00]-[09:02], [09:43]-[09:45] Marcos/Diego |
| PRD-CENARIO-03 | docs/PRD.md | Cenário de Uso | Endpoint do cliente fica fora do ar e a notificação é reentregue automaticamente | TRANSCRICAO | [09:14]-[09:18] Diego |
| PRD-CENARIO-04 | docs/PRD.md | Cenário de Uso | Cliente consulta histórico de envios para investigar notificação não recebida | TRANSCRICAO | [09:34] Marcos |
| PRD-CENARIO-05 | docs/PRD.md | Cenário de Uso | Cliente solicita rotação de secret por suspeita de vazamento | TRANSCRICAO | [09:21] Sofia |
| PRD-CENARIO-06 | docs/PRD.md | Cenário de Uso | Administrador reprocessa manualmente evento que esgotou tentativas automáticas | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência de notificação abaixo de 10 segundos | TRANSCRICAO | [09:01]-[09:02] Bruno/Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Entrega da feature até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Resiliência: até 5 tentativas / ~15h de janela de retry | TRANSCRICAO | [09:15]-[09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Reduzir a dependência de polling manual em `GET /orders` | TRANSCRICAO | [09:00] Marcos |
| PRD-ESCOPO-OUT-01 | docs/PRD.md | Restrição | Notificação proativa por e-mail — fora de escopo, adiada | TRANSCRICAO | [09:37]-[09:38] Marcos/Larissa |
| PRD-ESCOPO-OUT-02 | docs/PRD.md | Restrição | Rate limiting de envio — fora de escopo, observar depois | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| PRD-ESCOPO-OUT-03 | docs/PRD.md | Restrição | Dashboard visual — fora de escopo, projeto de frontend separado | TRANSCRICAO | [09:39]-[09:40] Marcos/Larissa |
| PRD-ESCOPO-OUT-04 | docs/PRD.md | Restrição | Garantia exactly-once — descartada | TRANSCRICAO | [09:25] Diego |
| PRD-ESCOPO-OUT-05 | docs/PRD.md | Restrição | Escalonamento multi-worker com ordenação global — fora de escopo | TRANSCRICAO | [09:12]-[09:13] Diego |
| PRD-ESCOPO-OUT-06 | docs/PRD.md | Restrição | Arquivamento automático da outbox — fora de escopo desta fase | TRANSCRICAO | [09:08] Diego |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de webhook (URL, filtro de status), secret gerada e devolvida na criação | TRANSCRICAO | [09:31]-[09:32] Marcos/Bruno |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Edição de webhook (URL, filtro de status, estado ativo) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listagem dos webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção do evento | TRANSCRICAO | [09:33]-[09:34] Marcos/Diego/Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico dos últimos 100 envios de um webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Notificação assinada com HMAC-SHA256 | TRANSCRICAO | [09:19]-[09:20] Sofia |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Identificador único de evento (`X-Event-Id`) para dedup (at-least-once) | TRANSCRICAO | [09:24]-[09:25] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Reentrega automática com backoff exponencial e DLQ após esgotar tentativas | TRANSCRICAO | [09:15]-[09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reprocessamento manual de DLQ por administrador, com auditoria | TRANSCRICAO | [09:18], [09:35]-[09:36] Diego/Sofia/Larissa |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Rejeição de cadastro de webhook com URL não-HTTPS | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação abaixo de 10 segundos na maioria dos casos | TRANSCRICAO | [09:01]-[09:02], [09:09]-[09:10] Bruno/Marcos/Diego/Larissa |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Sem infraestrutura nova; reaproveita MySQL/Prisma existentes | TRANSCRICAO | [09:07] Diego/Larissa |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | Eventos sobrevivem a até ~15h de indisponibilidade do cliente | TRANSCRICAO | [09:16]-[09:17] Diego |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Secret exclusiva por endpoint e TLS obrigatório | TRANSCRICAO | [09:20]-[09:23] Sofia |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64KB, falha em vez de truncar | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego/Larissa |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Worker roda separado da API e sobrevive a reinícios de deploy | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Registro do evento nunca dessincroniza da mudança de status (tudo-ou-nada) | TRANSCRICAO | [09:04]-[09:06], [09:40]-[09:41] Bruno/Diego |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Reprocessamento manual de DLQ registra o usuário responsável | TRANSCRICAO | [09:36] Sofia |
| PRD-DEC-01 | docs/PRD.md | Trade-off | Outbox transacional vs. disparo síncrono/fila externa | TRANSCRICAO | [09:06] Diego |
| PRD-DEC-02 | docs/PRD.md | Trade-off | Worker dedicado em polling de 2s vs. notificação reativa | TRANSCRICAO | [09:09] Diego |
| PRD-DEC-03 | docs/PRD.md | Trade-off | 5 tentativas de retry com DLQ vs. 3 tentativas ou retry indefinido | TRANSCRICAO | [09:17] Diego |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Secret por endpoint com rotação vs. secret global | TRANSCRICAO | [09:21] Sofia |
| PRD-DEC-05 | docs/PRD.md | Trade-off | At-least-once com dedup no cliente vs. exactly-once | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-06 | docs/PRD.md | Trade-off | Reuso máximo dos padrões do OMS vs. convenções novas para o módulo | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-07 | docs/PRD.md | Trade-off | Ordenação apenas por pedido (single-worker) vs. ordenação global desde já | TRANSCRICAO | [09:13] Diego |
| PRD-DEP-01 | docs/PRD.md | Dependência | Infraestrutura de dados existente (MySQL/Prisma), sem infraestrutura nova | TRANSCRICAO | [09:07] Diego/Larissa |
| PRD-DEP-02 | docs/PRD.md | Dependência | Padrões de código reaproveitados (módulos, AppError, logger, requireRole) | TRANSCRICAO | [09:27]-[09:30] Bruno/Diego/Larissa |
| PRD-DEP-03 | docs/PRD.md | Dependência | Novo processo de runtime (worker) na topologia de deploy | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-04 | docs/PRD.md | Dependência | Revisão de segurança dedicada de 2 dias antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-05 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor sobre at-least-once/dedup | TRANSCRICAO | [09:26], [09:36] Marcos |
| PRD-DEP-06 | docs/PRD.md | Dependência | Confirmação de prazo de fim de novembro com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-RISCO-01 | docs/PRD.md | Risco | Cliente indisponível por período prolongado — mitigado por retry+DLQ | TRANSCRICAO | [09:16]-[09:18] Diego |
| PRD-RISCO-02 | docs/PRD.md | Risco | Vazamento de secret de cliente — mitigado por secret por endpoint + rotação | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| PRD-RISCO-03 | docs/PRD.md | Risco | Atraso na entrega leva Atlas a migrar para concorrente | TRANSCRICAO | [09:00], [09:45]-[09:47] Marcos/Larissa |
| PRD-RISCO-04 | docs/PRD.md | Risco | Outbox cresce sem mecanismo de arquivamento nesta fase | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Cliente recebe eventos duplicados e não deduplica corretamente | TRANSCRICAO | [09:24]-[09:26] Diego/Sofia/Marcos |
| PRD-CA-01 | docs/PRD.md | Critério de Aceitação | Cliente cadastra webhook e recebe secret na resposta da criação | TRANSCRICAO | [09:31]-[09:32] Marcos/Bruno |
| PRD-CA-02 | docs/PRD.md | Critério de Aceitação | Notificação assinada entregue em <10s na maioria dos casos | TRANSCRICAO | [09:01]-[09:02], [09:09]-[09:10] Bruno/Marcos/Diego |
| PRD-CA-03 | docs/PRD.md | Critério de Aceitação | Cliente valida autenticidade via HMAC-SHA256 | TRANSCRICAO | [09:19]-[09:20] Sofia |
| PRD-CA-04 | docs/PRD.md | Critério de Aceitação | Cliente consulta histórico dos últimos 100 envios | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-05 | docs/PRD.md | Critério de Aceitação | Indisponibilidade do cliente não causa perda de evento (retry até ~15h) | TRANSCRICAO | [09:15]-[09:18] Diego |
| PRD-CA-06 | docs/PRD.md | Critério de Aceitação | Replay manual de DLQ por administrador é registrado para auditoria | TRANSCRICAO | [09:35]-[09:36] Sofia/Larissa |
| PRD-CA-07 | docs/PRD.md | Critério de Aceitação | Rotação de secret não interrompe validação durante grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-08 | docs/PRD.md | Critério de Aceitação | Cadastro de webhook com URL não-HTTPS é rejeitado | TRANSCRICAO | [09:23] Sofia |
| PRD-TESTE-01 | docs/PRD.md | Restrição | Testes automatizados seguem a infraestrutura já existente (`tests/`, `vitest.config.ts`) | CODIGO | tests/setup.ts |
| PRD-TESTE-02 | docs/PRD.md | Critério de Aceitação | Teste de atomicidade: falha na outbox provoca rollback da mudança de status | TRANSCRICAO | [09:40]-[09:41] Bruno/Diego |
| PRD-TESTE-03 | docs/PRD.md | Critério de Aceitação | Teste do worker: timeout, backoff completo e transição para DLQ | TRANSCRICAO | [09:15]-[09:18], [09:42] Diego |
| PRD-TESTE-04 | docs/PRD.md | Critério de Aceitação | Teste de HMAC e do grace period de rotação de secret | TRANSCRICAO | [09:20]-[09:22] Sofia |
| PRD-TESTE-05 | docs/PRD.md | Critério de Aceitação | Teste de RBAC do endpoint de replay (somente `ADMIN`) | TRANSCRICAO | [09:35]-[09:36] Sofia/Larissa |
| PRD-TESTE-06 | docs/PRD.md | Dependência | Revisão de segurança dedicada da Sofia como critério de saída antes do deploy | TRANSCRICAO | [09:46] Sofia |

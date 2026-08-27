# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-08-26 |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |

> Larissa se comprometeu a abrir o documento de design da feature e marcar uma sessão de revisão com Bruno e Diego antes do início da implementação (`[09:50] Larissa`). Este RFC é esse documento, submetido também à revisão de Marcos e Sofia.

## Resumo executivo (TL;DR)

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para serem notificados em tempo real quando o status dos seus pedidos muda, em vez de continuarem fazendo polling em `GET /orders` (`[09:00] Marcos`). Propomos um sistema de webhooks outbound: quando o status de um pedido muda, um evento é gravado atomicamente em uma tabela de outbox (mesma transação do `changeStatus`) e um worker dedicado, rodando em processo separado, entrega esse evento via HTTP ao endpoint cadastrado pelo cliente, com assinatura HMAC-SHA256, retry com backoff exponencial e Dead Letter Queue para falhas permanentes. A entrega é garantida como at-least-once, com deduplicação do lado do cliente via `X-Event-Id`. A solução reaproveita integralmente os padrões arquiteturais já estabelecidos no OMS (módulos, tratamento de erros, logger, RBAC).

## Contexto e problema

Hoje, clientes B2B integrados à plataforma descobrem mudanças de status de pedido fazendo polling periódico em `GET /orders`, o que Marcos descreveu como uma integração "lenta e cara" do lado deles (`[09:00] Marcos`). Três clientes pediram formalmente notificação em tempo real; a Atlas Comercial chegou a sinalizar que poderia migrar para um concorrente se a entrega não acontecer até o fim do trimestre (`[09:00] Marcos`). Quando questionados sobre o que significa "tempo real", os clientes indicaram que qualquer latência abaixo de 10 segundos já atende — o requisito central não é instantaneidade absoluta, e sim eliminar o polling manual (`[09:01]-[09:02] Bruno/Marcos`). O escopo é estritamente outbound: a plataforma envia notificações para os clientes, eles não enviam nada de volta (`[09:02] Sofia/Marcos`). O prazo confirmado com a Atlas é fim de novembro, e a estimativa da equipe é de 3 sprints, incluindo a revisão de segurança da Sofia (`[09:45]-[09:46] Marcos/Larissa`).

## Proposta técnica

A proposta se apoia em sete decisões arquiteturais, cada uma detalhada em um ADR próprio (ver seção "Decisões relacionadas"). Em visão geral:

- **Persistência do evento via padrão outbox:** ao mudar o status de um pedido, a mesma transação Prisma do `changeStatus` insere uma linha em uma tabela `webhook_outbox`, garantindo atomicidade entre a mudança de estado do pedido e o registro do evento a ser notificado — sem chamadas HTTP síncronas dentro da transação de negócio (ver **[ADR-001](adrs/ADR-001-outbox-no-mysql.md)**).
- **Worker dedicado em processo separado:** um processo Node independente (`src/worker.ts`), com seu próprio `PrismaClient`, faz polling da tabela de outbox a cada 2 segundos e dispara as entregas HTTP, isolado do ciclo de vida da API (ver **[ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)**).
- **Resiliência via retry e Dead Letter Queue:** falhas de entrega são reprocessadas com backoff exponencial (1m/5m/30m/2h/12h, 5 tentativas); esgotadas as tentativas, o evento vai para uma tabela `webhook_dead_letter`, com reprocessamento manual via endpoint administrativo (ver **[ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md)**).
- **Autenticidade via HMAC-SHA256:** cada evento é assinado com uma secret exclusiva por endpoint de webhook cadastrado, com suporte a rotação e grace period de 24h (ver **[ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)**).
- **Garantia de entrega at-least-once:** cada evento carrega um `X-Event-Id` único, permitindo que o cliente deduplique entregas repetidas do lado dele (ver **[ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)**).
- **Reuso máximo dos padrões existentes do OMS:** o novo módulo `src/modules/webhooks` segue a mesma estrutura, hierarquia de erros, logger e middlewares já usados pelos demais módulos da aplicação (ver **[ADR-006](adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md)**).
- **Ordenação apenas por pedido, não global:** com um único worker processando por ordem de inserção, a ordem dos eventos de um mesmo pedido é preservada, mas não há garantia de ordenação entre pedidos diferentes (ver **[ADR-007](adrs/ADR-007-ordenacao-por-order-id-sem-garantia-global.md)**).

Do ponto de vista funcional, a proposta também cobre CRUD de configuração de webhook por cliente (URL, eventos de interesse, secret gerada na criação), consulta de histórico de entregas, e um endpoint administrativo de replay de itens em DLQ restrito à role `ADMIN` (`[09:31]-[09:36] Marcos/Bruno/Sofia/Larissa`). O detalhamento de contratos, payloads, códigos de erro e fluxos passo a passo fica no FDD, não neste RFC.

## Alternativas consideradas

- **Disparo síncrono da notificação dentro da transação de `changeStatus`.** Descartada porque a transação de mudança de status já é pesada (atualiza pedido, histórico e estoque), e uma chamada HTTP síncrona a um cliente lento travaria a mudança de status de outros pedidos; além disso, não haveria uma estratégia coerente de rollback caso o cliente estivesse fora do ar (`[09:04] Bruno`).
- **Fila externa gerenciada (ex.: Redis Streams) em vez de outbox no MySQL.** Descartada por exigir a operação de infraestrutura adicional (ex.: um cluster Redis) para um time pequeno, sendo considerada overengineering frente ao volume esperado — o MySQL já usado pela aplicação resolve o problema sem infraestrutura nova (`[09:07] Diego/Larissa`).
- **Garantia de entrega exactly-once.** Descartada por exigir coordenação bilateral entre plataforma e cliente para confirmar processamento único, aumentando substancialmente a complexidade do sistema para resolver um problema que at-least-once com deduplicação por `X-Event-Id` já cobre na prática — o mesmo trade-off aceito por provedores de referência como Stripe e GitHub (`[09:25] Diego`).
- **Secret HMAC única e global para toda a plataforma.** Descartada porque um único vazamento comprometeria a autenticidade de eventos para todos os clientes simultaneamente; uma secret por endpoint contém o raio de impacto a um único cliente (`[09:21] Sofia`).

## Questões em aberto

- **Rate limiting de envio ao cliente.** Se um cliente tiver dezenas de pedidos mudando de status no mesmo minuto, a plataforma pode acabar bombardeando o endpoint dele com chamadas simultâneas. O time reconheceu o risco, mas decidiu não incluir controle de taxa nesta fase — a orientação foi observar o comportamento em produção e decidir depois se isso vira um problema real (`[09:38]-[09:39] Diego/Larissa`).
- **Notificação proativa de falha ao cliente.** Marcos perguntou se seria possível avisar o cliente (por exemplo, por e-mail) quando o webhook dele estiver falhando repetidamente. Larissa decidiu que isso fica fora do escopo desta fase, ficando como candidato para uma fase futura, após medir o impacto da primeira versão (`[09:37]-[09:38] Marcos/Larissa`).
- **Escalonamento do worker para múltiplos processos.** A garantia de ordenação por `order_id` (ver ADR-007) depende de um único worker ativo. Diego levantou que, se for necessário escalar para múltiplos workers no futuro, será preciso introduzir particionamento por `order_id` ou lock pessimista — mas isso foi explicitamente classificado como "problema do futuro, não agora" e não tem desenho definido (`[09:13] Diego`).

## Impacto e riscos

**Impacto de negócio:** a entrega tempestiva desta feature está diretamente ligada à retenção de três clientes B2B relevantes, com prazo confirmado de fim de novembro junto à Atlas Comercial; atraso ou não entrega carrega risco explícito de churn (`[09:00] Marcos`, `[09:45] Marcos`).

**Impacto de engenharia:** introduz um novo módulo de aplicação e um novo processo de runtime (o worker) a ser deployado, monitorado e operado continuamente, além de duas tabelas novas (outbox e DLQ). A estimativa da equipe é de 3 sprints, já incluindo a revisão de segurança da Sofia (`[09:45]-[09:46] Larissa`).

**Impacto de segurança:** a feature cria uma nova superfície de saída, com chamadas HTTP para URLs de terceiros e a necessidade de gerar, armazenar e rotacionar secrets. Sofia solicitou reservar ao menos dois dias úteis de revisão de segurança dedicada, com atenção especial a HMAC e geração de secret, antes do deploy (`[09:46] Sofia`).

**Riscos principais:**

| Risco | Mitigação |
| --- | --- |
| Endpoint do cliente fica indisponível por horas e eventos se perdem | Retry com backoff exponencial de até ~15h e DLQ com reprocessamento manual (`[09:15]-[09:18] Diego`, ver ADR-003) |
| Secret de um cliente vaza (já ocorreu antes) e compromete a integração | Secret exclusiva por endpoint, com rotação e grace period de 24h (`[09:21]-[09:22] Sofia/Diego`, ver ADR-004) |
| Tabela de outbox cresce indefinidamente e degrada performance de leitura | Índices em status e `created_at`; arquivamento de eventos entregues após ~30 dias, fora do escopo desta fase de implementação (`[09:08] Diego`, ver ADR-001) |
| Worker único vira gargalo de vazão à medida que o volume cresce | Aceito como limitação conhecida por ora; escalonamento requer redesenho de particionamento (`[09:12]-[09:13] Diego`, ver ADR-007) |

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker de webhooks em processo separado com polling de 2 segundos](adrs/ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial e Dead Letter Queue](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md)
- [ADR-004 — Autenticação de webhooks via HMAC-SHA256 com secret por endpoint](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Garantia de entrega at-least-once com idempotência via X-Event-Id](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões arquiteturais existentes do OMS no módulo de webhooks](adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md)
- [ADR-007 — Ordenação implícita por order_id, sem garantia de ordenação global](adrs/ADR-007-ordenacao-por-order-id-sem-garantia-global.md)

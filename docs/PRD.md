# PRD — Sistema de Webhooks de Notificação de Pedidos

> Este é o documento de mais alto nível do pacote e consolida as decisões já detalhadas no [RFC](RFC.md), nas [ADRs](adrs/) e no [FDD](FDD.md). Aqui o foco é "por que e o quê"; "como construir" fica no FDD.

## Resumo e contexto da feature

Hoje, clientes B2B integrados à plataforma descobrem mudanças de status dos seus pedidos fazendo polling periódico em `GET /orders`. Três clientes — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para serem notificados em tempo real (`[09:00] Marcos`). A feature proposta é um sistema de webhooks outbound: sempre que o status de um pedido muda, a plataforma envia uma notificação HTTP assinada para o(s) endpoint(s) cadastrado(s) pelo cliente, com garantia de entrega e proteção contra falhas de rede ou indisponibilidade do lado do cliente.

## Problema e motivação

Marcos relatou que o polling atual deixa a integração "lenta e cara" para os clientes (`[09:00] Marcos`). A Atlas Comercial sinalizou que pode migrar para um concorrente se a entrega não acontecer até o fim do trimestre (`[09:00] Marcos`), o que caracteriza risco real de churn de conta estratégica. Quando questionados sobre o que significa "tempo real", os clientes indicaram que qualquer latência abaixo de 10 segundos já resolve — o problema central não é milissegundos, é eliminar a necessidade de ficar consultando manualmente (`[09:01]-[09:02] Bruno/Marcos`).

## Público-alvo e cenários de uso

**Público-alvo:** sistemas de clientes B2B integrados à plataforma via API, representados inicialmente pelos três clientes que solicitaram a feature (Atlas Comercial, MaxDistribuição, Nova Cargo) (`[09:00] Marcos`). O consumo é sempre outbound — a plataforma envia, o cliente recebe; não há webhooks de entrada (`[09:02] Sofia/Marcos`).

**Cenários de uso principais:**
- Um cliente cadastra um endpoint de webhook, informando a URL do sistema dele e os status de pedido que quer acompanhar (`[09:31]-[09:34] Marcos/Bruno`).
- Um pedido do cliente muda de status (ex.: para `SHIPPED`) e o sistema dele recebe automaticamente uma notificação HTTP assinada, sem precisar consultar `GET /orders` (`[09:00]-[09:02], [09:43]-[09:45] Marcos/Diego`).
- O endpoint do cliente fica temporariamente fora do ar; a plataforma reentrega a notificação automaticamente até o serviço dele voltar, sem intervenção manual (`[09:14]-[09:18] Diego`).
- Um cliente quer investigar por que não recebeu uma notificação e consulta o histórico dos últimos envios daquele webhook (`[09:34] Marcos`).
- Um cliente suspeita que sua secret vazou e solicita rotação pela API, sem interromper a validação das notificações durante a transição (`[09:21] Sofia`).
- Um administrador da plataforma reprocessa manualmente um evento que esgotou todas as tentativas automáticas de entrega (`[09:18] Diego`).

## Objetivos e métricas de sucesso

| Objetivo | Métrica / Meta | Fonte |
| --- | --- | --- |
| Notificar o cliente em tempo hábil, eliminando a necessidade de polling manual | Latência de notificação **abaixo de 10 segundos** na maioria dos casos (limiar definido pelos próprios clientes como equivalente a "tempo real") | `[09:01]-[09:02] Bruno/Marcos` |
| Entregar a feature dentro do prazo comercial acordado com a Atlas | Disponível até **o fim de novembro** | `[09:45] Marcos` |
| Garantir resiliência de entrega sem perda de eventos em indisponibilidades temporárias do cliente | Até **5 tentativas** de reentrega, cobrindo uma janela de aproximadamente **15 horas** antes de mover para falha permanente | `[09:15]-[09:17] Diego` |
| Reduzir a dependência dos clientes B2B de polling manual em `GET /orders` | Objetivo qualitativo — sem meta numérica de redução de chamadas declarada na reunião | `[09:00] Marcos` |

## Escopo

### Incluso no escopo

- Cadastro, edição, remoção e listagem de webhooks por cliente, com filtro de status de interesse (`[09:31]-[09:34] Marcos/Bruno/Diego`).
- Geração de secret na criação do webhook e rotação de secret via API com grace period de 24h (`[09:21] Sofia`).
- Entrega assinada via HMAC-SHA256, com TLS obrigatório na URL cadastrada (`[09:19]-[09:23] Sofia`).
- Garantia de entrega at-least-once, com `X-Event-Id` único por evento para deduplicação do lado do cliente (`[09:24]-[09:26] Diego`).
- Retry com backoff exponencial e Dead Letter Queue (DLQ) para falhas permanentes, com reprocessamento manual restrito a `ADMIN` (`[09:15]-[09:18], [09:35]-[09:36] Diego/Sofia/Larissa`).
- Consulta do histórico de entregas de um webhook (`[09:34] Marcos`).
- Persistência do evento via padrão outbox, na mesma transação da mudança de status do pedido (`[09:06], [09:40]-[09:41] Diego/Bruno`).
- Worker dedicado em processo separado, com polling de 2 segundos (`[09:08]-[09:11] Diego`).

### Fora de escopo

- **Notificação proativa ao cliente em caso de falhas recorrentes (ex.: e-mail).** Adiado explicitamente para uma fase futura, após medição do impacto da primeira versão (`[09:37]-[09:38] Marcos/Larissa`).
- **Rate limiting de envio por cliente.** Levantado como risco (cliente com muitos pedidos mudando de status pode ser "bombardeado" com chamadas), mas decidido não implementar agora — a orientação foi observar em produção e decidir depois (`[09:38]-[09:39] Diego/Larissa`).
- **Dashboard visual para o cliente acompanhar seus webhooks.** Explicitamente descartado desta fase; ficaria a cargo de um projeto separado do time de frontend (`[09:39]-[09:40] Marcos/Larissa`).
- **Garantia de entrega exactly-once.** Descartada por exigir coordenação bilateral complexa e desproporcional ao ganho frente a at-least-once com deduplicação (`[09:25] Diego`).
- **Escalonamento do worker para múltiplos processos com ordenação global garantida.** Classificado como "problema do futuro, não agora"; a versão atual opera com um único worker (`[09:12]-[09:13] Diego`).
- **Arquivamento automático de eventos já entregues na tabela de outbox.** Mencionado como necessário eventualmente (após ~30 dias), mas fora do escopo de implementação desta fase (`[09:08] Diego`).

## Requisitos funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RF-01 | Cliente pode cadastrar um webhook informando URL e a lista de status de pedido de interesse; a plataforma gera e devolve a secret na resposta da criação | `[09:31]-[09:32] Marcos/Bruno` |
| RF-02 | Cliente pode editar um webhook cadastrado (URL, filtro de status, estado ativo) | `[09:33] Bruno` |
| RF-03 | Cliente pode remover um webhook cadastrado | `[09:33] Bruno` |
| RF-04 | Cliente pode listar os webhooks cadastrados para ele | `[09:33] Bruno` |
| RF-05 | Cada webhook só recebe notificações dos status que escolheu (filtro de eventos aplicado na inserção do evento, não no envio) | `[09:33]-[09:34] Marcos/Diego/Bruno` |
| RF-06 | Cliente pode consultar o histórico dos últimos 100 envios de um webhook, com sucesso/falha, payload, resposta e tempo de resposta | `[09:34] Marcos` |
| RF-07 | Cliente pode rotacionar a secret do seu webhook pela API; a secret antiga permanece válida por 24h em paralelo à nova | `[09:21] Sofia` |
| RF-08 | Toda notificação é assinada com HMAC-SHA256, permitindo ao cliente validar autenticidade e integridade | `[09:19]-[09:20] Sofia` |
| RF-09 | Toda notificação carrega um identificador único de evento (`X-Event-Id`) para o cliente deduplicar reentregas (garantia at-least-once) | `[09:24]-[09:25] Diego` |
| RF-10 | Falhas de entrega são reenviadas automaticamente com backoff exponencial; esgotadas as tentativas, o evento vai para uma fila de falhas permanentes (DLQ) | `[09:15]-[09:18] Diego` |
| RF-11 | Administrador pode reprocessar manualmente um evento em falha permanente, com essa ação registrada para auditoria | `[09:18], [09:35]-[09:36] Diego/Sofia/Larissa` |
| RF-12 | A plataforma rejeita o cadastro de um webhook cuja URL não use HTTPS | `[09:23] Sofia` |

## Requisitos não funcionais

| ID | Requisito | Fonte |
| --- | --- | --- |
| RNF-01 | Latência de notificação abaixo de 10 segundos na maioria dos casos | `[09:01]-[09:02], [09:09]-[09:10] Bruno/Marcos/Diego/Larissa` |
| RNF-02 | A solução não introduz infraestrutura nova (ex.: filas externas como Redis); reaproveita o MySQL/Prisma já existentes | `[09:07] Diego/Larissa` |
| RNF-03 | Eventos sobrevivem a até ~15 horas de indisponibilidade do endpoint do cliente antes de serem considerados falha permanente | `[09:16]-[09:17] Diego` |
| RNF-04 | Cada endpoint de webhook tem uma secret exclusiva (não global) e comunicação exige TLS | `[09:20]-[09:23] Sofia` |
| RNF-05 | Payload de evento limitado a 64KB; eventos maiores falham em vez de serem truncados | `[09:23]-[09:24] Sofia/Diego/Larissa` |
| RNF-06 | O processo responsável pela entrega (worker) roda separado da API e sobrevive a reinícios de deploy dela | `[09:11] Diego` |
| RNF-07 | O registro do evento de notificação nunca fica dessincronizado da mudança de status do pedido (garantia tudo-ou-nada) | `[09:04]-[09:06], [09:40]-[09:41] Bruno/Diego` |
| RNF-08 | Toda ação de reprocessamento manual de DLQ registra o usuário responsável, para auditoria | `[09:36] Sofia` |

## Decisões e trade-offs principais

As decisões abaixo estão detalhadas cada uma em sua respectiva ADR; aqui ficam resumidas apenas para consolidação:

- **Outbox transacional em vez de disparo síncrono ou fila externa** — trade-off: complexidade de operar um worker próprio, em troca de atomicidade garantida sem infraestrutura nova (ver [ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
- **Worker dedicado em polling de 2s em vez de notificação reativa** — trade-off: piso de latência de 2s no pior caso, em troca de simplicidade (MySQL não suporta `LISTEN`/`NOTIFY`) (ver [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)).
- **5 tentativas de retry com DLQ em vez de 3 ou indefinido** — trade-off: aceitar até ~15h de latência de recuperação em favor de cobrir indisponibilidades reais já observadas, sem eventos pendurados para sempre (ver [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dead-letter-queue.md)).
- **Secret por endpoint com rotação em vez de secret global** — trade-off: mais complexidade de gestão de secrets, em troca de conter o impacto de um vazamento a um único cliente (ver [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)).
- **At-least-once com dedup no cliente em vez de exactly-once** — trade-off: transfere responsabilidade de deduplicação ao cliente, em troca de evitar a complexidade de coordenação bilateral (ver [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).
- **Reuso máximo dos padrões do OMS em vez de convenções novas para o módulo** — trade-off: adaptar padrões pensados para request/response HTTP a um worker assíncrono, em troca de consistência e menor risco de regressão (ver [ADR-006](adrs/ADR-006-reuso-dos-padroes-arquiteturais-existentes.md)).
- **Ordenação apenas por pedido (single-worker) em vez de ordenação global desde já** — trade-off: limita o escalonamento futuro do worker, em troca de simplicidade imediata e atendimento à necessidade real dos clientes (ver [ADR-007](adrs/ADR-007-ordenacao-por-order-id-sem-garantia-global.md)).

## Dependências

- **Infraestrutura de dados existente:** MySQL via Prisma, já provisionado pela aplicação (`prisma/schema.prisma`) — nenhuma peça de infraestrutura nova é necessária (`[09:07] Diego/Larissa`).
- **Padrões e componentes de código reaproveitados:** módulo por domínio, `AppError`/hierarquia de erros, logger Pino, `error.middleware.ts`, `requireRole`/`authenticate` (`[09:27]-[09:30] Bruno/Diego/Larissa`, ver ADR-006).
- **Novo processo de runtime:** um processo worker (`src/worker.ts`) precisa ser adicionado à topologia de deploy, rodando em paralelo à API (`[09:11] Diego`).
- **Revisão de segurança dedicada:** pelo menos dois dias úteis de revisão da Sofia antes do deploy, focada em HMAC e geração de secret — pré-condição para o lançamento (`[09:46] Sofia`).
- **Comunicação com o cliente:** Marcos é responsável por documentar o comportamento de at-least-once/deduplicação e o funcionamento geral da integração no portal do desenvolvedor (`[09:26], [09:36] Marcos`).
- **Confirmação de prazo:** depende de Marcos confirmar e alinhar o prazo de fim de novembro diretamente com a Atlas (`[09:47] Marcos`).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação | Fonte |
| --- | --- | --- | --- | --- |
| Endpoint do cliente fica indisponível por um período prolongado (ex.: manutenção planejada) e o evento se perde | Média — já ocorreu antes com um cliente (indisponibilidade de 2h) | Alto — cliente perde notificação de mudança de status de pedido | Retry com backoff exponencial cobrindo ~15h + DLQ com reprocessamento manual | `[09:16]-[09:18] Diego` |
| Secret de um cliente vaza (ex.: em log de aplicação do lado dele) | Média — já ocorreu antes com um cliente | Alto — compromete a autenticidade das notificações daquele cliente | Secret exclusiva por endpoint (não global) + rotação com grace period de 24h | `[09:21]-[09:22] Sofia/Diego` |
| Atraso na entrega da feature leva a Atlas Comercial a migrar para um concorrente | Baixa a Média — depende do cumprimento do prazo estimado | Alto — perda de cliente B2B estratégico | Estimativa de 3 sprints já incluindo a revisão de segurança, alinhada com o PM para confirmação junto ao cliente | `[09:00], [09:45]-[09:47] Marcos/Larissa` |
| Tabela de outbox cresce continuamente sem mecanismo de arquivamento implementado nesta fase | Média — arquivamento ficou fora do escopo desta entrega | Médio — possível degradação de performance de leitura do worker ao longo do tempo | Índices em status e `created_at`; arquivamento tratado como débito técnico conhecido para fase futura | `[09:08] Diego` |
| Cliente recebe eventos duplicados (garantia at-least-once) e não implementa deduplicação corretamente do lado dele | Média — depende da maturidade da integração de cada cliente | Médio — pode gerar processamento duplicado no sistema do cliente | `X-Event-Id` único por evento, documentado de forma destacada no portal do desenvolvedor | `[09:24]-[09:26] Diego/Sofia/Marcos` |

## Critérios de aceitação

- Um cliente consegue cadastrar um webhook informando URL e status de interesse, e recebe a secret gerada na resposta da criação.
- Ao mudar o status de um pedido para um valor de interesse de um webhook ativo daquele cliente, uma notificação HTTP assinada é entregue em menos de 10 segundos na maioria dos casos.
- O cliente consegue validar a autenticidade da notificação recebida usando a assinatura HMAC-SHA256 e a secret que possui.
- O cliente consegue consultar o histórico dos últimos 100 envios de um webhook, incluindo sucesso/falha e tempo de resposta.
- Uma indisponibilidade temporária do endpoint do cliente não causa perda do evento: a notificação é reentregue automaticamente por até ~15 horas antes de cair em falha permanente (DLQ).
- Um administrador consegue reprocessar manualmente um evento em falha permanente, com essa ação registrada para auditoria.
- O cliente consegue rotacionar sua secret sem que as notificações deixem de validar durante a janela de transição de 24h.
- Não é possível cadastrar um webhook com URL que não use HTTPS.

## Estratégia de testes e validação

- **Testes automatizados:** seguir a infraestrutura de testes já existente no projeto (`tests/`, `vitest.config.ts`, execução sequencial contra um MySQL real), cobrindo o novo módulo de webhooks (service/repository) nos mesmos moldes dos módulos já testados.
- **Teste de atomicidade:** validar que uma falha ao inserir o evento na `webhook_outbox` provoca rollback completo da mudança de status do pedido, e que uma mudança de status bem-sucedida sempre resulta no evento correspondente sendo registrado (`[09:40]-[09:41] Bruno/Diego`).
- **Teste do worker e da resiliência:** simular entrega com sucesso, timeout de 10s, progressão completa do backoff exponencial e transição para DLQ após a 5ª falha (`[09:15]-[09:18], [09:42] Diego`).
- **Teste de segurança da assinatura:** validar geração e verificação de HMAC-SHA256, incluindo o cenário de dupla validade de secret durante o grace period de 24h de uma rotação (`[09:20]-[09:22] Sofia`).
- **Teste de controle de acesso:** confirmar que o endpoint de replay de DLQ só é executável por usuários com role `ADMIN` (`[09:35]-[09:36] Sofia/Larissa`).
- **Revisão de segurança dedicada:** reservar ao menos dois dias úteis de revisão da Sofia antes do deploy, com foco em HMAC e geração de secret, como critério de saída antes do lançamento (`[09:46] Sofia`).

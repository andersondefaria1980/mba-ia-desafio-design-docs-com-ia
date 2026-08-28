# ADR-002: Worker de webhooks em processo separado com polling de 2 segundos

## Status

Aceito

## Contexto

Uma vez decidido o padrão outbox (ADR-001), era preciso definir como o worker consumidor leria a tabela `webhook_outbox`. Diego propôs polling em loop, buscando a cada 2 segundos os eventos pendentes mais antigos (`[09:09] Diego`).

Bruno questionou se não seria possível usar um trigger de banco para ser mais reativo. Diego explicou que o MySQL não tem um mecanismo nativo equivalente ao `NOTIFY`/`LISTEN` do PostgreSQL: um trigger existe, mas só executa SQL, não notifica um processo externo; fazer isso funcionar exigiria soluções improvisadas como escrever em arquivo ou chamar um endpoint, o que foi considerado inadequado (`[09:09] Diego`). O polling de 2 segundos atende com folga o requisito de latência dos clientes B2B (Atlas, MaxDistribuição, Nova Cargo), que consideram "tempo real" qualquer coisa abaixo de 10 segundos (`[09:02] Marcos`, `[09:09]-[09:10] Diego/Marcos`).

Diego também levantou que o worker precisa rodar como processo separado da API, e não dentro da mesma instância: se a API reiniciar, o worker não pode ser perdido junto (`[09:11] Diego`). Larissa sugeriu seguir o padrão de entry point já existente no projeto — hoje a API sobe via `src/server.ts` — criando um `src/worker.ts` novo e um script `npm run worker` (`[09:11] Larissa`). Confirmado no código: `src/server.ts` já é um entry point dedicado que constrói a aplicação, sobe o listener HTTP e trata `SIGINT`/`SIGTERM` para shutdown gracioso, separado da montagem da aplicação em si — um padrão replicável para o worker.

Bruno e Diego concordaram que o worker precisa se conectar ao mesmo banco, mas como `PrismaClient` é por processo, o worker deve abrir sua própria instância — mesma `DATABASE_URL`, mesma stack, processo Node diferente (`[09:11] Bruno`, `[09:29]-[09:30] Diego/Bruno`).

## Decisão

O worker de processamento da outbox roda como um processo Node independente (`src/worker.ts`, iniciado via `npm run worker`), com sua própria instância de `PrismaClient` conectada à mesma `DATABASE_URL` da API. O worker opera em loop de polling, verificando a tabela `webhook_outbox` a cada 2 segundos, processando os eventos pendentes mais antigos em lote pequeno e marcando-os como entregues (`[09:08]-[09:10] Diego`).

## Alternativas Consideradas

- **Notificação reativa via trigger de banco.** Rejeitada: MySQL não oferece um mecanismo nativo de notificação de processo externo (como o `LISTEN`/`NOTIFY` do Postgres); a alternativa exigiria workarounds artificiais (escrita em arquivo, chamada a endpoint) consideradas inadequadas (`[09:09] Diego`).
- **Rodar o processamento dentro do mesmo processo da API.** Rejeitada: acoplaria o ciclo de vida do worker ao da API — um restart da API derrubaria o worker junto (`[09:11] Diego`).

## Consequências

**Positivas:**
- Desacopla o ciclo de vida do worker do ciclo de vida da API: reinícios de deploy da API não interrompem o processamento de eventos pendentes (`[09:11] Diego`).
- Atende com folga o requisito de latência percebida pelos clientes (< 10s), com um piso de 2s no pior caso (`[09:10] Larissa`: "a latência mínima vai ser 2 segundos no pior caso. Aceitamos.").
- Reaproveita o padrão de entry point de processo já estabelecido pelo projeto (`src/server.ts`), reduzindo a superfície de decisões novas de infraestrutura.

**Negativas:**
- Introduz um processo adicional a ser deployado, monitorado e escalado separadamente da API.
- Polling é menos reativo que uma notificação push: sempre existe uma latência mínima de até 2 segundos, ainda que aceitável para o requisito atual (`[09:10] Larissa`).

# ADR-001: Padrão Outbox no MySQL para entrega de eventos de webhook

## Status

Aceito

## Contexto

A feature de Webhooks de Notificação de Pedidos precisa disparar uma notificação HTTP para sistemas de clientes sempre que o status de um pedido muda. A primeira pergunta levantada na reunião foi se esse disparo deveria ser síncrono, dentro do próprio serviço de pedidos, ou assíncrono via algum mecanismo de fila (`[09:03] Larissa`).

Bruno argumentou contra o disparo síncrono: a transação de mudança de status já é pesada — ela atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` dos produtos do pedido — e acrescentar uma chamada HTTP nesse meio faria um cliente lento travar a mudança de status de outros pedidos (`[09:04] Bruno`). Bruno também apontou que, se o endpoint do cliente estivesse fora do ar, não haveria como fazer rollback coerente da mudança de status (`[09:04] Bruno`).

Essa transação já existe hoje em `src/modules/orders/order.service.ts`, método `changeStatus()` (linhas 126–179), que roda dentro de um único `this.prisma.$transaction(...)`, atualizando `Order.status`, inserindo em `OrderStatusHistory` e ajustando `Product.stockQuantity` — confirmado no código.

Diego propôs o padrão outbox: dentro da mesma transação SQL que atualiza `orders` e `order_status_history`, inserir também uma linha em uma tabela `webhook_outbox` com o evento; um worker separado lê essa tabela e dispara as chamadas HTTP (`[09:06] Diego`). Isso garante que, se a transação principal comitar, o evento foi registrado, e se der rollback, o evento some junto — sem inconsistência possível (`[09:06] Diego`).

A alternativa de usar uma fila externa como Redis Streams foi levantada e descartada: exigiria subir infraestrutura nova para um time pequeno (`[09:07] Larissa`, `[09:07] Diego`).

## Decisão

Adotar o padrão outbox: uma tabela `webhook_outbox` no mesmo banco MySQL usado pela aplicação, com inserção de eventos feita **dentro da mesma transação Prisma** do `changeStatus()` (`src/modules/orders/order.service.ts`). A tabela terá índice em status de processamento (pendente/processando/falhou/entregue) e em `created_at` para leitura eficiente pelo worker (`[09:08] Diego`). Linhas já entregues serão arquivadas após ~30 dias, mas esse mecanismo de arquivamento fica fora do escopo desta decisão (`[09:08] Diego`).

## Alternativas Consideradas

- **Chamada HTTP síncrona dentro da transação de `changeStatus`.** Rejeitada: bloquearia a transação de mudança de status em caso de cliente lento e não tem estratégia de rollback quando o cliente está indisponível (`[09:04] Bruno`).
- **Fila externa (ex.: Redis Streams).** Rejeitada: exigiria subir e operar infraestrutura nova (ex.: Redis Cluster) para um time pequeno, sendo considerado overengineering frente ao volume esperado (`[09:07] Diego`).

## Consequências

**Positivas:**
- Atomicidade garantida entre a mudança de status e o registro do evento de webhook: não existe caso de status mudar sem o evento ser registrado, nem evento registrado sem a mudança ter sido commitada (`[09:06] Diego`, `[09:41] Diego`: "Se ficar fora da transação, perde a garantia toda").
- Não introduz nova peça de infraestrutura: reaproveita o MySQL já operado pelo time (`[09:07] Diego`).

**Negativas:**
- Exige um processo worker separado para drenar a tabela (ver ADR-002), diferente de uma fila gerenciada que já entregaria isso pronto.
- A tabela cresce continuamente e precisa de rotina de arquivamento após entrega, mecanismo esse deixado fora do escopo desta decisão (`[09:08] Diego`).

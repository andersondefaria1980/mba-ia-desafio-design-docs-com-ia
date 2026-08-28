# ADR-007: Ordenação implícita por order_id, sem garantia de ordenação global

## Status

Aceito

## Contexto

Com o worker em polling definido (ADR-002), Larissa levantou uma questão de consistência: se um pedido X muda de PAID para PROCESSING e depois para SHIPPED em sequência rápida, o cliente recebe os eventos na ordem correta? (`[09:12] Larissa`)

Diego explicou que isso depende do número de workers ativos: com um único worker, o processamento segue a ordem de `created_at` da tabela de outbox, então o cliente recebe os eventos de um mesmo pedido na ordem certa. Se no futuro a plataforma escalar para múltiplos workers em paralelo, essa garantia se perde (`[09:12] Diego`). Por ora, a decisão foi manter um único worker, com ordenação implícita apenas por `order_id` (`[09:12] Diego`).

Bruno perguntou o que aconteceria se um dia fosse necessário escalar. Diego respondeu que nesse cenário seria possível particionar por `order_id` ou usar lock pessimista, mas classificou isso como "problema do futuro, não agora" (`[09:13] Diego`). Larissa determinou que essa limitação fosse documentada: não há garantia de ordenação global, apenas por `order_id`, e apenas enquanto o sistema operar com um único worker (`[09:13] Larissa`). Marcos confirmou que os clientes nunca pediram garantia de ordenação global — eles só querem saber quando cada pedido deles individualmente muda de status (`[09:14] Marcos`).

## Decisão

O sistema garante ordenação de eventos apenas por `order_id`, como consequência de operar com um único worker processando a outbox em ordem de `created_at` (ver ADR-002). Não há garantia de ordenação global entre pedidos diferentes. Escalonamento para múltiplos workers em paralelo é uma decisão arquitetural futura, condicionada à introdução de particionamento por `order_id` ou lock pessimista.

## Alternativas Consideradas

- **Particionar a outbox por `order_id` ou usar lock pessimista desde já, para já suportar múltiplos workers em paralelo.** Rejeitada/adiada: os clientes não solicitaram garantia de ordenação global, apenas correção por pedido individual (`[09:14] Marcos`), e a complexidade adicional não se justifica frente à necessidade atual (`[09:13] Diego`: "isso é problema do futuro, não agora").

## Consequências

**Positivas:**
- Solução simples de implementar agora, sem custo de coordenação entre workers, e suficiente para o requisito real dos clientes (ordem correta por pedido individual) (`[09:14] Marcos`).
- A limitação é conhecida e documentada desde a decisão inicial, evitando expectativas erradas sobre garantias de ordenação globais.

**Negativas:**
- Limita a capacidade de escalar o throughput do worker horizontalmente sem antes revisitar essa decisão: introduzir múltiplos workers exigirá particionamento por `order_id` ou lock pessimista, que hoje não existe (`[09:13] Diego`).
- Cria um único ponto de processamento (um worker) como fator limitante de vazão, ainda que suficiente para o volume e a latência exigidos hoje.

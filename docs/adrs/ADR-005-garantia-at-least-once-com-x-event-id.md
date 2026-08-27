# ADR-005: Garantia de entrega at-least-once com idempotência via X-Event-Id

## Status

Aceito

## Contexto

Definidos o mecanismo de retry e a DLQ (ADR-003), foi preciso decidir qual a semântica de entrega garantida ao cliente. Diego declarou que a plataforma garantirá at-least-once: pode acontecer de o cliente receber o mesmo evento mais de uma vez, e ele precisa estar preparado para isso (`[09:24] Diego`).

Para permitir a deduplicação do lado do cliente, foi decidido enviar um header `X-Event-Id`, contendo um UUID gerado no momento em que o evento entra na tabela de outbox, único por evento — se o cliente recebe o mesmo evento duas vezes, ele deduplica localmente usando esse identificador (`[09:25] Diego`).

Sofia observou que essa abordagem transfere responsabilidade para o cliente (`[09:25] Sofia`). Diego reconheceu o trade-off, mas defendeu a escolha como padrão de mercado, citando Stripe e GitHub como exemplos de provedores que adotam a mesma estratégia; garantir exactly-once exigiria coordenação entre os dois lados e aumentaria consideravelmente a complexidade, para resolver um problema que at-least-once com `event_id` já cobre na prática (`[09:25] Diego`). Marcos se comprometeu a documentar isso de forma destacada no portal do desenvolvedor para os clientes (`[09:26] Marcos`).

## Decisão

A plataforma garante entrega at-least-once para eventos de webhook. Cada evento carrega um `X-Event-Id` (UUID) gerado no momento da inserção na outbox, permitindo que o cliente implemente deduplicação do lado dele em caso de reenvio.

## Alternativas Consideradas

- **Garantia de entrega exactly-once.** Rejeitada: exigiria coordenação bilateral entre a plataforma e cada cliente para confirmar processamento único, aumentando substancialmente a complexidade do sistema para um ganho marginal frente a at-least-once com deduplicação por `event_id` (`[09:25] Diego`).

## Consequências

**Positivas:**
- Mantém o sistema de entrega mais simples, sem exigir protocolos de confirmação bilateral entre plataforma e cliente.
- Segue um padrão já validado por provedores de referência do mercado (Stripe, GitHub), o que facilita a adoção pelos clientes B2B (`[09:25] Diego`).
- O `X-Event-Id` também serve como identificador único para rastreamento/observabilidade dos próprios reenvios internos.

**Negativas:**
- Transfere para o cliente a responsabilidade de implementar deduplicação, o que exige documentação clara e destacada para evitar efeitos colaterais de processamento duplicado do lado dele (`[09:25]-[09:26] Sofia/Marcos`).

# ADR-004: Autenticação de webhooks via HMAC-SHA256 com secret por endpoint

## Status

Aceito

## Contexto

A feature envia eventos com dados de pedidos para endpoints fora da infraestrutura da empresa. Sofia, engenheira de segurança, apontou que o cliente precisa conseguir validar que a requisição realmente veio da plataforma e que o payload não foi adulterado no caminho (`[09:19] Sofia`). O padrão escolhido foi HMAC: assinar o payload com uma secret compartilhada entre a plataforma e o cliente, enviando a assinatura em um header (`X-Signature`), que o cliente verifica do seu lado (`[09:20] Sofia`). O algoritmo definido foi HMAC-SHA256, por ser padrão de mercado com suporte em praticamente qualquer biblioteca cliente (`[09:20] Sofia`).

Sofia também exigiu que cada endpoint de webhook cadastrado por um cliente tenha uma secret única, e não uma secret global da plataforma — se uma secret vazar, o impacto fica restrito àquele endpoint, em vez de comprometer todos os clientes (`[09:21] Sofia`). Isso implica que a tabela de configuração de webhook armazene `url`, `secret`, `customer_id` e um estado de ativação por registro (`[09:21] Bruno`).

Foi definido também suporte a rotação de secret: o cliente pode solicitar uma nova secret via API, e a secret antiga permanece válida em paralelo por 24 horas, dando tempo para o cliente migrar seus sistemas antes que a antiga deixe de funcionar (`[09:21] Sofia`). Diego reforçou a relevância dessa capacidade citando um incidente real: já houve cliente que vazou uma secret em log de aplicação (`[09:22] Diego`).

Duas exigências de segurança adicionais discutidas na mesma reunião — TLS obrigatório no cadastro da URL do webhook e limite de 64KB no tamanho do payload — foram classificadas pelo próprio grupo como validação de schema/requisito não funcional, e não como decisão arquitetural separada (`[09:23]-[09:24] Sofia/Diego/Larissa`), permanecendo fora do escopo desta ADR.

## Decisão

Toda entrega de webhook é assinada com HMAC-SHA256 sobre o corpo da requisição. Cada endpoint de webhook cadastrado por um cliente possui sua própria secret (não há secret global da plataforma), armazenada junto com `url`, `customer_id` e estado de ativação. A secret é rotacionável via API; durante a rotação, a secret antiga permanece válida por um grace period de 24 horas em paralelo com a nova.

## Alternativas Consideradas

- **Secret única e global da plataforma para todos os clientes.** Rejeitada: um único vazamento comprometeria a autenticidade de eventos para todos os clientes simultaneamente, em vez de um único endpoint (`[09:21] Sofia`).

## Consequências

**Positivas:**
- Garante autenticidade e integridade do payload usando um padrão amplamente adotado no mercado, com suporte trivial do lado do cliente (`[09:20] Sofia`).
- Contém o raio de impacto de um vazamento de secret a um único endpoint/cliente, em vez de comprometer toda a base (`[09:21] Sofia`).
- A rotação com grace period de 24h permite resposta operacional a incidentes reais de vazamento sem quebrar a integração do cliente durante a transição (`[09:21]-[09:22] Sofia/Diego`).

**Negativas:**
- Adiciona complexidade operacional: é preciso validar contra duas secrets simultaneamente durante o período de rotação, além de gerar, armazenar e expor a secret de forma segura.
- Transfere para o cliente a responsabilidade de implementar corretamente a verificação HMAC do lado dele, exigindo documentação clara (a cargo do time de produto no portal do desenvolvedor).

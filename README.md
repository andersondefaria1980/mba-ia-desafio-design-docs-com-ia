# Sistema de Webhooks de Notificação de Pedidos — Design Docs

Pacote de design docs (PRD, RFC, FDD, ADRs, Tracker) produzido para a feature de Webhooks de Notificação de Pedidos do OMS, a partir da transcrição de uma reunião técnica (`TRANSCRICAO.md`) e do código-fonte existente da aplicação.

## Sobre o desafio

O desafio original pedia para transformar a transcrição de uma reunião técnica — na qual tech lead, PM, dois engenheiros e uma engenheira de segurança fecham o desenho de um sistema de webhooks outbound para o OMS — em um pacote completo de design docs, usando IA como ferramenta principal de produção. O ponto central não era gerar documentação bonita, e sim rastreável: cada requisito, decisão ou restrição registrado nos documentos precisa remontar a um timestamp específico da transcrição ou a um caminho real de arquivo no código, sem inventar nada que não tenha sido de fato discutido ou que não exista de fato na base.

O enunciado completo do desafio está no repositório base do curso: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Ferramentas de IA utilizadas

- **Claude Code** — única ferramenta usada em toda a produção do pacote. Papéis desempenhados:
  - Leitura integral da transcrição (`TRANSCRICAO.md`) e extração dirigida de decisões fechadas, requisitos, restrições e pontos descartados/adiados.
  - Exploração do código-fonte (via subagentes de exploração, com acesso somente-leitura) para confirmar a existência e o comportamento real dos caminhos de arquivo citados nos documentos, antes de qualquer um deles ser escrito.
  - Redação de cada documento (ADRs, RFC, FDD, PRD, Tracker) em Markdown, seguindo o formato exigido pelo desafio.
  - Validação cruzada de cada documento contra a transcrição e o código, incluindo uma auditoria final "de conjunto" de todo o pacote.

## Workflow adotado

A produção seguiu a ordem sugerida pelo próprio enunciado do desafio, por ser a que faz cada documento se apoiar no anterior em vez de repetir conteúdo:

1. **Leitura e mapeamento inicial** — leitura completa da transcrição e verificação, um a um, de que os caminhos de código citados na documentação de contexto do projeto (`order.service.ts`, `order.status.ts`, `app-error.ts`, `http-errors.ts`, `error.middleware.ts`, `auth.middleware.ts`, `logger/index.ts`, `prisma/schema.prisma`, entre outros) existem e se comportam como descrito, e de que a feature é de fato greenfield (nenhuma menção prévia a webhook/outbox/fila no código).
2. **ADRs primeiro** — as 7 decisões arquiteturais foram extraídas e escritas antes de qualquer outro documento, por formarem o esqueleto técnico de tudo que vem depois.
3. **RFC** — consolidação da proposta técnica em cima das ADRs já prontas, com alternativas descartadas e questões em aberto linkando diretamente para elas.
4. **FDD** — o desenho de implementação (fluxos, contratos HTTP, matriz de erros, integração com o código existente) construído sobre as decisões já fechadas no RFC/ADRs.
5. **PRD** — produzido por último entre os documentos grandes, como consolidação de mais alto nível do que já estava decidido tecnicamente.
6. **Tracker** — montado ao final, varrendo os quatro documentos e as sete ADRs para mapear cada item identificável à sua origem.
7. **README** (este arquivo) — escrito por último, documentando o processo já concluído.

Cada documento passou por uma rodada de geração seguida de uma rodada de validação dirigida (não uma simples releitura): checagem de seções obrigatórias, contagem de itens mínimos exigidos pelos critérios de aceite (ex.: ≥8 requisitos funcionais no PRD, ≥4 endpoints com request/response no FDD), e confirmação de que nenhum item descartado na reunião apareceu como requisito fechado.

## Prompts customizados

Prompt usado para extrair decisões e requisitos da transcrição, dirigido a distinguir decisão fechada de ideia descartada/adiada (em vez de um prompt genérico "resuma a reunião"):

```
Leia TRANSCRICAO.md do início ao fim. Para cada tópico técnico discutido,
classifique em uma de três categorias:

1. DECISÃO FECHADA — o grupo chegou a um consenso explícito (cite o
   timestamp [hh:mm] e o nome de quem fechou a decisão).
2. DESCARTADA — foi levantada e rejeitada explicitamente, com o motivo
   do descarte (cite o timestamp de quem descartou).
3. ADIADA / EM ABERTO — foi levantada mas não decidida, ou explicitamente
   empurrada para uma fase futura (cite o timestamp).

Não infira decisões que não foram verbalizadas. Se o mesmo tópico for
retomado depois na reunião com uma decisão diferente, sinalize a
divergência em vez de escolher uma versão silenciosamente.
```

Prompt usado para a validação cruzada de cada documento antes de considerá-lo pronto (aplicado a cada um de PRD/RFC/FDD/ADRs/Tracker):

```
Você acabou de gerar docs/<DOCUMENTO>.md. Antes de aceitar como pronto,
faça uma auditoria adversarial:

1. Para cada afirmação de requisito, decisão ou restrição, encontre a
   citação [hh:mm] Nome em TRANSCRICAO.md ou o caminho de arquivo real
   no código que a sustenta. Se não encontrar, marque como suspeita de
   alucinação e proponha remoção ou correção.
2. Confira contra a lista de critérios de aceite deste documento no
   DESAFIO.md (seções obrigatórias, contagens mínimas).
3. Liste explicitamente qualquer item da reunião que foi descartado ou
   adiado e que apareça aqui como se fosse requisito fechado.

Reporte os problemas encontrados antes de qualquer correção — não
corrija silenciosamente.
```

## Iterações e ajustes

O processo não saiu correto na primeira geração em pelo menos três pontos concretos, todos pegos pela rodada de validação dirigida em vez de aceitos de primeira:

1. **FDD — contratos de endpoint incompletos.** A primeira versão da seção "Contratos públicos" não tinha request **e** response lado a lado para todos os 7 endpoints (alguns endpoints sem corpo de requisição, como `DELETE` e o replay administrativo, ficaram sem nota explicando a ausência). Corrigido adicionando notas de "sem corpo" onde aplicável, para deixar claro que a omissão era intencional e não uma lacuna de documentação.
2. **Tracker — cobertura incompleta do PRD.** A primeira versão do Tracker cobria Requisitos Funcionais, Não Funcionais, Riscos e Critérios de Aceitação do PRD, mas deixou de fora inteiramente a seção "Público-alvo e cenários de uso" (7 itens). A rodada de validação de cobertura (contagem de itens identificáveis vs. linhas no tracker) pegou a lacuna, e as linhas `PRD-PUBLICO-01` e `PRD-CENARIO-01` a `06` foram adicionadas.
3. **FDD — tensão entre duas falas sobre o mesmo código de erro.** Na transcrição, Sofia trata a validação de URL HTTPS como "só uma validação Zod" (`[09:23] Sofia`), enquanto Bruno, minutos depois, lista `WEBHOOK_INVALID_URL` como um código de erro do módulo (`[09:28] Bruno`). Em vez de escolher uma das duas falas e ignorar a outra, a matriz de erros do FDD documenta as duas camadas explicitamente: a checagem básica de HTTPS acontece no schema Zod (gerando `VALIDATION_ERROR` genérico), e `WEBHOOK_INVALID_URL` fica reservado para falhas de validação de URL em nível de domínio além do que o Zod cobre (ex.: revalidação em `PATCH`). Essa reconciliação foi sinalizada explicitamente antes de prosseguir, por não ser uma dedução direta de uma única fala.

## Como navegar a entrega

Ordem sugerida de leitura, da mais alta para a mais baixa altura de abstração:

1. [`docs/PRD.md`](docs/PRD.md) — por que a feature existe, para quem, e o que significa sucesso.
2. [`docs/RFC.md`](docs/RFC.md) — a proposta técnica em nível de arquitetura, com alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/) — as 7 decisões arquiteturais individuais que sustentam o RFC (`ADR-001` a `ADR-007`, cada uma com contexto, decisão, alternativas e consequências).
4. [`docs/FDD.md`](docs/FDD.md) — o desenho de implementação: modelo de dados, fluxos, contratos HTTP, matriz de erros e a seção "Integração com o sistema existente".
5. [`docs/TRACKER.md`](docs/TRACKER.md) — para verificar a origem (transcrição ou código) de qualquer afirmação específica dos quatro documentos acima.
6. [`TRANSCRICAO.md`](TRANSCRICAO.md) — a fonte primária, para quem quiser conferir uma citação no contexto original da conversa.

O código da aplicação (`src/`, `prisma/`, `tests/`) não foi alterado nesta entrega — serve apenas como contexto e referência para os documentos acima.

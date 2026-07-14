# Design Docs — Sistema de Webhooks de Notificação de Pedidos

Pacote de documentação técnica gerado a partir da transcrição da reunião e do código existente do OMS (Order Management System).

> Enunciado original do desafio: [mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

## Sobre o desafio

A partir de uma reunião técnica já ocorrida (registrada só em `TRANSCRICAO.md`) e de uma aplicação Node.js + TypeScript funcional sem notificações externas, a tarefa foi produzir um pacote acionável de design docs — PRD, RFC, FDD, ADRs e Tracker — para o time de engenharia iniciar a implementação de **webhooks outbound** de mudança de status de pedidos.

A restrição central foi não inventar requisitos: tudo registrado precisa rastrear à transcrição ou ao código. Também foi obrigatório separar alturas de documento (produto × arquitetura × decisão pontual × implementação) e explicitar o que a reunião **descartou** ou **adiou**.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Cursor (Agent / Composer)** | Leitura do repositório, exploração de `src/` e Prisma, análise da transcrição, redação e refinamentos dos Markdown, commits incrementais. |
| **Critério humano (maestro)** | Definição da ordem de produção, prompts dirigidos (“o que entra / o que não entra”), checagem de contradicão com a call e checklist de aceite do desafio. |

Nenhuma alteração foi feita em `src/`, `prisma/` ou `tests/` — o código serviu apenas de contexto.

## Workflow adotado

Ordem alinhada à sugestão do desafio (decisões primeiro):

1. **Contextualização** — leitura de `README.md` (enunciado), `TRANSCRICAO.md` completa e pontos de integração no código (`order.service`, erros, auth, logger, schema).
2. **ADRs (6)** — formalizar as decisões fechadas da call (esqueleto do “como”).
3. **RFC** — consolidar proposta, alternativas descartadas e questões em aberto; linkar ADRs.
4. **FDD** — detalhar fluxos, contratos HTTP, erros `WEBHOOK_*`, resiliência, observabilidade e integração com arquivos reais.
5. **PRD** — consolidar visão de produto, FRs/NFRs, métricas, fora de escopo e riscos.
6. **Tracker** — varredura cruzada documento → timestamp/arquivo.
7. **README do processo** — este arquivo.
8. **Commits incrementais** por artefato para facilitar revisão.

## Prompts customizados

Dois prompts (adaptados) que direcionaram a produção — genéricos do tipo “gere um PRD” foram evitados de propósito.

### Prompt 1 — Extração e classificação da reunião

```text
Você é um analista de design docs de sistema legado/OMS.
Com base APENAS em TRANSCRICAO.md:

1) Liste decisões FECHADAS (com timestamp + falante).
2) Liste requisitos funcionais explícitos.
3) Liste itens DESCARTADOS ou ADIADOS (fora de escopo).
4) Liste questões EM ABERTO (não decididas).
5) Liste ganchos com código existente mencionados (arquivos/padrões).

Regras:
- Não invente nada que não esteja na fala.
- Separe claramente "decidido" vs "mencionado mas adiado".
- Para cada item, cite [hh:mm] Nome.
```

### Prompt 2 — FDD ancorado no código real

```text
Produza docs/FDD.md para webhooks outbound do OMS.
Fontes permitidas: TRANSCRICAO.md + arquivos existentes do repo.
É proibido inventar requisitos sem origem.

Obrigatório:
- Fluxos: outbox na TX, worker polling, retry/backoff, DLQ/replay.
- ≥4 endpoints HTTP com request/response de exemplo e status codes.
- Matriz de erros com prefixo WEBHOOK_.
- Seção "Integração com o sistema existente" citando ≥4 caminhos REAIS
  (ex.: src/modules/orders/order.service.ts, auth.middleware.ts, etc.)
  e COMO o módulo se integra a cada um.
- Observabilidade: logs (Pino), métricas derivadas do desenho da outbox/DLQ,
  correlação via event_id — sem inventar stack de APM não citada.

Não duplique o nível de detalhe do RFC; foque em "como construir".
```

## Iterações e ajustes

Foram necessários vários ciclos de geração/revisão. Principais correções:

1. **Primeira passagem genérica vs. “o que NÃO entra”**  
   Rascunhos iniciais tendiam a tratar rate limiting, e-mail e dashboard como requisitos da fase 1. Correção: promover esses itens explicitamente a **fora de escopo / questões em aberto**, como a call fechou ([09:37–09:40]).

2. **Fronteira RFC × FDD**  
   A primeira consolidação técnica misturava payloads completos e matriz de erros no RFC. Ajuste: RFC ficou em 2–4 “páginas” (abordagem + alternativas + abertos + links ADR); detalhe de contrato foi empurrado para o FDD.

3. **Rastreabilidade e anti-alucinação**  
   Ao montar o Tracker, qualquer frase sem timestamp ou arquivo correspondente era candidata a remoção/reescrita (ex.: não afirmar OpenTelemetry/APM se a reunião só firmou Pino + auditoria de replay + `X-Event-Id`).

4. **Integração com código**  
   Conferência manual de caminhos (`order.service.ts`, `auth.middleware.ts`, `error.middleware.ts`, `app-error.ts`, `schema.prisma`, etc.) para garantir que nenhum arquivo citado é inexistente — critério de aceite de consistência geral.

5. **Referência cruzada final (checklist literal)**  
   Ainda faltavam: `PRD-NFR-01` alinhado no Tracker (antes `PRD-NFR-LAT`), `PRD-OOS-05` (multi-worker), headings `## Metadados` / `## Status`, e marcar `src/worker.ts` como **a criar** (não como arquivo já existente).

**Iterações principais até o pacote atual:** ~5 ciclos (extração → ADRs/RFC → FDD/PRD → Tracker/README → checklist cruzada de aceite).

## Como navegar a entrega

Leitura sugerida:

| Ordem | Arquivo | Por quê |
| --- | --- | --- |
| 1 | [`TRANSCRICAO.md`](./TRANSCRICAO.md) | Fonte primária da verdade da reunião |
| 2 | [`docs/adrs/README.md`](./docs/adrs/README.md) | Índice das decisões fechadas |
| 3 | [`docs/RFC.md`](./docs/RFC.md) | Proposta técnica para review |
| 4 | [`docs/FDD.md`](./docs/FDD.md) | Especificação para começar a codar |
| 5 | [`docs/PRD.md`](./docs/PRD.md) | Problema, escopo e métricas de produto |
| 6 | [`docs/TRACKER.md`](./docs/TRACKER.md) | De onde veio cada item |

```
.
├── README.md                 ← este processo
├── TRANSCRICAO.md            ← não alterar
└── docs/
    ├── PRD.md
    ├── RFC.md
    ├── FDD.md
    ├── TRACKER.md
    └── adrs/
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-retry-backoff-dlq.md
        ├── ADR-003-autenticacao-hmac-sha256.md
        ├── ADR-004-garantia-at-least-once.md
        ├── ADR-005-worker-polling-processo-separado.md
        └── ADR-006-reuso-padroes-projeto.md
```

Código da aplicação (`src/`, `prisma/`, `tests/`) permanece intocado e serve apenas como referência de integração.

---

# Aprendizado

Notas do que ficou claro ao produzir este pacote com IA a partir da transcrição e do código legado — e do que a referência cruzada final ainda revelou.

## Sobre o papel de cada documento

- **PRD** responde *por que / o quê* (problema, escopo, métricas). Não deve detalhar retry intervals nem shape de header.
- **RFC** responde *como pretendemos resolver* e deixa abertos; é documento de review, não manual de implementação.
- **ADR** congela *uma* decisão com trade-off. Se a call fechou seis decisões centrais, seis ADRs deixam o esqueleto explícito.
- **FDD** é o “pegar e codar”: fluxos, contratos, erros, integração com arquivos reais.
- **Tracker** não é documento de mercado; é rede de segurança contra alucinação. Se não há `[hh:mm] Nome` ou caminho de arquivo, o item provavelmente não deveria existir.

Duplicar o mesmo nível de detalhe entre RFC e FDD foi o erro mais fácil de cometer — e o mais importante de corrigir.

## O que a reunião ensinou (além do que ela “aprovou”)

Identificar o que **não** entra importa tanto quanto o que entra:

- E-mail de alerta, dashboard, rate limiting de saída e multi-worker foram mencionados e **explicitamente** adiados.
- Prompt genérico (“gere o PRD da feature”) puxa esses itens de volta para o escopo. Prompt dirigido pedindo “descartados / adiados / em aberto” com timestamp evita isso.

## Outbox + código existente

- A garantia de consistência não é “chamar o webhook depois”; é inserir na outbox **dentro** da mesma `$transaction` de `changeStatus` (`order.service.ts`).
- Síncrono HTTP na TX foi descartado com razão clara: disponibilidade do cliente não pode definir o destino do pedido.
- Reusar `AppError`, Pino, Zod e `requireRole` não é estética: reduz superfície nova e encaixa no error middleware que já existe.

## At-least-once na prática

- Retries + timeout + replay de DLQ **implicam** entrega duplicada possível.
- `X-Event-Id` empurra idempotência para o cliente; documentar isso no portal (PM) é requisito de produto, não detalhe opcional.

## Uso de IA neste desafio

- Ordem boa: explorar código/transcrição → ADRs → RFC → FDD → PRD → Tracker → README.
- Commits por artefato ajudam a revisar e reverter um documento sem misturar tudo.
- A IA acelera redação; o valor do humano está em classificar falas, recusar invenção e amarrar cada afirmação a uma origem.

## Referência cruzada final (checklist de aceite)

Na varredura documento ↔ enunciado ↔ Tracker, o conteúdo já cobria os mínimos — mas ainda havia buracos de **consistência**:

- **IDs desalinhados**: o PRD tinha `NFR-01`, o Tracker tinha `PRD-NFR-LAT`. Se o ID não bate, o cruzamento falha mesmo com o conteúdo certo.
- **Fora de escopo incompleto no Tracker**: multi-worker estava no PRD e faltava linha (`PRD-OOS-05`).
- **Checklist literal**: informações presentes sem o heading exigido (`## Metadados` no RFC, `## Status` nos ADRs) podem falhar em leitura automática do enunciado.
- **Arquivo citado ≠ arquivo existente**: `src/worker.ts` é decisão da call, mas **ainda não existe** no repo. Em docs, marcar como “a criar” e usar caminhos reais só na coluna `CODIGO`.
- **Cobertura medida > sensação**: quantificar % `TRANSCRICAO`/`CODIGO` no Tracker (aqui ≈83% / 15 linhas) evita achar que “maioria” basta.

## Checklist mental para próximos design docs

1. Separar decidido / adiado / em aberto antes de escrever prose.
2. Nomear arquivos reais de integração (não “o service de orders”); marcar explícito o que ainda será criado.
3. Preencher Tracker em paralelo; buraco na Localização = alarme; IDs iguais aos dos docs.
4. Revisar seções de “fora de escopo” com o mesmo rigor dos FRs — item a item no Tracker.
5. Antes do push: checklist de aceite do enunciado *literalmente* (headings + contagens), não só “conteúdo parece ok”.

# ADR-005: Worker em processo separado com polling

- **Status:** Aceito
- **Data:** derivado da reunião técnica documentada em `TRANSCRICAO.md`
- **Decisores:** Diego (Plataforma), Larissa (Tech Lead), Bruno (Pedidos), Marcos (PM)

## Contexto

Eventos na outbox precisam ser despachados sem acoplar o ciclo de vida ao processo da API HTTP. Clientes aceitam latência abaixo de **10 segundos** como “tempo real”. MySQL não oferece equivalente a `NOTIFY`/`LISTEN` do Postgres para acordar um processo externo.

## Decisão

1. Worker como **processo Node separado** (entry point `src/worker.ts`, script `npm run worker`), **não** embutido na mesma instância da API (`src/server.ts`).
2. Leitura da outbox por **polling a cada 2 segundos**: buscar eventos pendentes mais antigos em batch pequeno, processar, marcar.
3. Mesmo banco e mesma stack (Prisma), mas **nova instância de `PrismaClient` por processo**.
4. Fase 1: **single-worker**; ordenação implícita por `created_at` / `order_id`. Ordering global e multi-worker (partição por `order_id` ou lock pessimista) ficam como limitação conhecida / futuro.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
| --- | --- |
| Worker no mesmo processo da API | Restart da API derruba o worker. ([09:11] Diego) |
| Trigger MySQL “notificando” o worker | Trigger só executa SQL; improvisar arquivo/endpoint fica frágil; polling de 2s atende o SLO de &lt;10s. ([09:09] Bruno, Diego) |
| Multi-worker paralelo na fase 1 | Quebra garantia de ordenação por pedido; overkill agora. ([09:12–09:13] Diego, Bruno) |

## Consequências

**Positivas**

- Isolamento de falha/restart entre API e despacho.
- Latência típica compatível com requisito de negócio (&lt;10s) com poll de 2s.
- Modelo simples de operar na fase 1.

**Negativas / trade-offs**

- Latência mínima ligada ao intervalo de poll (~2s no pior caso ocioso).
- Polling gera carga periódica no MySQL (mitigada por índice + batch).
- Scaling horizontal do worker exige redesenho de ordering/locks.

# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | Documento produzido a partir da reunião técnica em `TRANSCRICAO.md` |
| **Revisores** | Marcos (PM), Bruno (Pedidos), Diego (Plataforma), Sofia (Segurança) |
| **Feature** | Outbound webhooks para mudança de status de pedidos |

## TL;DR

Propomos notificar clientes B2B sobre mudanças de status de pedido via **webhooks outbound**, usando **Transactional Outbox no MySQL**, **worker em processo separado com polling de 2s**, **retry com backoff + DLQ**, autenticação **HMAC-SHA256** (secret por endpoint) e entrega **at-least-once** com **`X-Event-Id`**. A feature reutiliza os padrões já estabelecidos do OMS (módulo, `AppError`, Pino, Zod, `requireRole`). Detalhe de implementação está no [FDD](./FDD.md); decisões pontuais nos [ADRs](./adrs/README.md).

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) hoje fazem polling em `GET /orders` para descobrir mudanças de status. Isso encarece a integração deles e já há pressão comercial (Atlas: risco de migração se não houver entrega até o fim do trimestre / novembro discutido na call).

Para os clientes, latência **abaixo de 10 segundos** já é “tempo real”. O escopo é **somente outbound** (nós → cliente). O OMS atual **não possui** mecanismos de notificação, fila ou eventos — o vácuo é preenchido por esta RFC.

A mudança de status (`OrderService.changeStatus`) já é uma transação pesada (pedido, histórico, estoque). Qualquer desenho não pode acoplar HTTP externo a essa transação.

## Proposta técnica

### Visão geral

```
[API Orders] --(mesma TX)--> [webhook_outbox]
                                    |
                              [Worker polling 2s]
                                    |
                         HTTP POST https://cliente...
                         (HMAC, X-Event-Id, timeout 10s)
                                    |
                    sucesso → delivered | falha → retry/backoff → DLQ
```

1. **Configuração**: CRUD de webhooks por `customer_id` (URL HTTPS, secret gerada, filtro de status, ativo/inativo), autenticação JWT; filtro aplicado **na inserção** na outbox.
2. **Publicação**: em `changeStatus`, função `publishWebhookEvent(tx, …)` grava evento com **payload snapshot** e UUID, só se existir webhook interessado naquele status.
3. **Despacho**: processo a criar `src/worker.ts` (+ lógica em `src/modules/webhooks/`) faz polling, envia HTTP com headers assinados, timeout 10s.
4. **Resiliência**: 5 tentativas (1m/5m/30m/2h/12h); depois `webhook_dead_letter`; replay admin `POST /admin/webhooks/dead-letter/:id/replay` com role `ADMIN` e log de auditoria.
5. **Observabilidade do cliente**: `GET /webhooks/:id/deliveries` (últimas entregas: sucesso/falha, payload, response, tempo).
6. **Segurança**: HMAC-SHA256, secret por endpoint, rotação com grace 24h, HTTPS only, payload ≤ 64KB.

### Limitações conhecidas (fase 1)

- Single-worker: ordenação por pedido enquanto houver um worker; sem garantia de ordering global.
- Sem rate limiting de saída (observar).
- Sem e-mail de alerta ao cliente; sem dashboard visual.

### Módulo e tipagem

Novo módulo `src/modules/webhooks` + entry worker; registros relacionados: [ADR-001](./adrs/ADR-001-outbox-no-mysql.md), [ADR-005](./adrs/ADR-005-worker-polling-processo-separado.md), [ADR-006](./adrs/ADR-006-reuso-padroes-projeto.md).

## Alternativas consideradas

| # | Alternativa | Trade-off que motivou o descarte | Origem |
| --- | --- | --- | --- |
| 1 | HTTP síncrono dentro de `changeStatus` | Cliente lento/offline bloqueia ou corrompe o fluxo de status; viola isolamento da transação de pedidos. | [09:03–09:04] Larissa, Bruno; [09:06] Diego |
| 2 | Redis Streams / fila externa | Nova infra operacional para time pequeno; overengineering vs. outbox no MySQL existente. | [09:07] Larissa, Diego |
| 3 | Trigger MySQL acordando o worker | Sem listener nativo tipo Postgres NOTIFY; improvisos frágeis; polling de 2s já atende &lt;10s. | [09:09] Bruno, Diego |
| 4 | Exactly-once | Complexidade de coordenação bilateral; mercado resolve com at-least-once + id de evento. | [09:25] Diego, Sofia |

(As duas primeiras são as alternativas estruturais principais à abordagem escolhida; 3 e 4 reforçam decisões conexas.)

## Questões em aberto

Pontos levantados na reunião e **não fechados** / adiados:

1. **Rate limiting de envio** para o endpoint do cliente (ex.: 50 mudanças/minuto). Decisão: observar em produção e implementar só se virar problema. ([09:38–09:39] Diego, Larissa)
2. **Alerta proativo ao cliente** (ex.: e-mail após N falhas consecutivas). Fora do escopo desta fase; candidato a fase seguinte. ([09:37–09:38] Marcos, Larissa)
3. **Endurecimento de autorização** no CRUD de webhooks (hoje: qualquer role autenticada). Softer agora; pode endurecer depois. ([09:36–09:37] Marcos, Sofia)
4. **Estratégia de escala multi-worker** (partição por `order_id` / lock pessimista) quando ordering single-worker deixar de bastar. Explicitamente “problema do futuro”. ([09:13] Diego)

Itens 1 e 2 são os mínimos obrigatórios desta seção; 3 e 4 registram outras aberturas da call.

## Impacto e riscos

| Impacto / risco | Notas |
| --- | --- |
| Código | Extensão crítica em `order.service.ts` (`changeStatus`); novo módulo + worker; migrations outbox/DLQ/config. |
| Operação | Processo `npm run worker` a manter no ar; DLQ a monitorar. |
| Segurança | Review obrigatória de Sofia (≥2 dias úteis) antes do deploy (HMAC/secret). |
| Produto | Prazo estimado: **3 sprints** (incluindo review de segurança). |
| Cliente | Precisa implementar verificação HMAC e dedup por `X-Event-Id`. |

Riscos detalhados com mitigação: ver [PRD](./PRD.md) e [FDD](./FDD.md).

## Decisões relacionadas

| ADR | Tema |
| --- | --- |
| [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) | Outbox no MySQL |
| [ADR-002](./adrs/ADR-002-retry-backoff-dlq.md) | Retry, backoff, DLQ e replay |
| [ADR-003](./adrs/ADR-003-autenticacao-hmac-sha256.md) | HMAC-SHA256 e secrets |
| [ADR-004](./adrs/ADR-004-garantia-at-least-once.md) | At-least-once e `X-Event-Id` |
| [ADR-005](./adrs/ADR-005-worker-polling-processo-separado.md) | Worker polling separado |
| [ADR-006](./adrs/ADR-006-reuso-padroes-projeto.md) | Reuso dos padrões do OMS |

## Próximos passos sugeridos

1. Revisão deste RFC pelos revisores listados.
2. Congelar ADRs aceitos; detalhar contratos no [FDD](./FDD.md).
3. Agendar review de segurança (Sofia) no fim da implementação.

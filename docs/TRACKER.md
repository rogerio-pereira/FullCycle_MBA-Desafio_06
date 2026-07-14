# Tracker de Rastreabilidade

Mapeamento dos itens dos design docs à origem em `TRANSCRICAO.md` ou no código do OMS.  
Se um item não tiver `Localização` preenchível, não deveria estar na documentação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pedem notificação | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Risco comercial | Atlas pode migrar se não houver entrega no trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência percebida &lt; 10s (poll worker 2s; “tempo real” na prática) | TRANSCRICAO | [09:02] Marcos; [09:09–09:10] Diego, Larissa |
| PRD-SCOPE-OUTBOUND | docs/PRD.md | Escopo | Webhooks somente outbound (nós → cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo / Métrica | Meta p95 &lt; 10s da commit à primeira tentativa | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo / Métrica | Prazo estimado de 3 sprints | TRANSCRICAO | [09:46] Larissa |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail de alerta por falhas consecutivas adiado | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Dashboard visual fora; só endpoints API | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Rate limiting de saída observar depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Arquivamento outbox 30 dias fora do escopo | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Multi-worker e ordering global adiados | TRANSCRICAO | [09:12–09:14] Diego, Bruno, Larissa |
| PRD-DEP-01 | docs/PRD.md | Dependência | Documentação de integração no portal (HMAC, dedup) — PM | TRANSCRICAO | [09:45] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | POST criar webhook; secret gerada; customerId no body/path | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | PATCH/DELETE/GET de configuração | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Filtro de eventos por lista de status | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Enfileirar evento na mesma TX de mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | POST outbound com payload enxuto e headers assinados | TRANSCRICAO | [09:43] Diego |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Retry 5× com backoff 1m/5m/30m/2h/12h + DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Replay DLQ admin com role ADMIN e auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | GET deliveries (sucesso/falha, payload, response, tempo) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | At-least-once com dedup por X-Event-Id | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Timeout HTTP 10s | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Limite payload 64KB com erro (não truncar) | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | TLS/HTTPS obrigatório | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 e secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Reuso AppError/Pino/middleware/códigos WEBHOOK_ | TRANSCRICAO | [09:29] Larissa |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Review segurança ≥2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-RISK-01 | docs/PRD.md | Risco | Cliente sem dedup processa duplicatas | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-02 | docs/PRD.md | Risco | Secret vazada no cliente (já ocorreu) | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Pressão de prazo Atlas / trimestre | TRANSCRICAO | [09:00] Marcos |
| RFC-TLDR-01 | docs/RFC.md | Decisão | Abordagem outbox + worker + HMAC + at-least-once | TRANSCRICAO | [09:48] Larissa |
| RFC-ALT-01 | docs/RFC.md | Trade-off | Descarte de HTTP síncrono em changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Trade-off | Descarte de Redis Streams / fila externa | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Trade-off | Descarte de trigger MySQL para acordar worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Trade-off | Descarte de exactly-once | TRANSCRICAO | [09:25] Diego |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída observar | TRANSCRICAO | [09:39] Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | E-mail de alerta em fase futura | TRANSCRICAO | [09:38] Marcos |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Endurecer auth do CRUD depois | TRANSCRICAO | [09:37] Sofia |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Multi-worker / partição por order_id no futuro | TRANSCRICAO | [09:13] Diego |
| RFC-IMP-01 | docs/RFC.md | Impacto | Review segurança Sofia como gate de deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Insert outbox na mesma TX de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | publishWebhookEvent(tx, …) sem injetar repo inteiro | TRANSCRICAO | [09:41] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Polling worker a cada 2s | TRANSCRICAO | [09:09] Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Backoff e movimentação para DLQ separada | TRANSCRICAO | [09:18] Diego |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| FDD-FLOW-06 | docs/FDD.md | Fluxo | Filtro de status na inserção (não no envio) | TRANSCRICAO | [09:34] Bruno |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks cria configuração e devolve secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET/PATCH/DELETE de webhooks | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | POST rotate-secret com grace 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | Payload event_type order.status_changed sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| FDD-ERR-01 | docs/FDD.md | Restrição | Prefixo WEBHOOK_ nos códigos de erro | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL / WEBHOOK_NOT_FOUND / etc. | TRANSCRICAO | [09:28] Bruno |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout 10s no HTTP do worker | TRANSCRICAO | [09:42] Diego |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logger Pino reutilizado; sem stack nova | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria em replay admin | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação via event_id / X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-INT-01 | docs/FDD.md | Integração | Extensão de changeStatus na TX existente | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Reuso da máquina de estados OrderStatus | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros via AppError / hierarquia existente | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Subclasses HTTP / InvalidStatusTransition etc. | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-05 | docs/FDD.md | Integração | Error middleware centralizado sem novo contrato | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | authenticate + requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Logger Pino compartilhado | CODIGO | src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | Entry API como referência para entry worker | CODIGO | src/server.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Registro de rotas no buildApiRouter | CODIGO | src/routes/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Models UUID / OrderStatus no schema Prisma | CODIGO | prisma/schema.prisma |
| FDD-INT-11 | docs/FDD.md | Integração | Paginação no padrão paginated() | CODIGO | src/shared/http/response.ts |
| FDD-LIM-01 | docs/FDD.md | Limitação | Ordering só por order_id com single-worker | TRANSCRICAO | [09:13] Larissa |
| FDD-LIM-02 | docs/FDD.md | Limitação | UUID (não autoincrement) na outbox | TRANSCRICAO | [09:51] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox transacional no MySQL | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Sync HTTP e Redis descartados | TRANSCRICAO | [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-backoff-dlq.md | Decisão | 5 retries + DLQ tabela separada + replay | TRANSCRICAO | [09:17] Larissa |
| ADR-002-ALT | docs/adrs/ADR-002-retry-backoff-dlq.md | Trade-off | 3 tentativas insuficientes vs manutenção 2h | TRANSCRICAO | [09:16] Diego |
| ADR-003 | docs/adrs/ADR-003-autenticacao-hmac-sha256.md | Decisão | HMAC-SHA256, secret por endpoint, grace 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-003-ALT | docs/adrs/ADR-003-autenticacao-hmac-sha256.md | Trade-off | Secret global descartada | TRANSCRICAO | [09:21] Sofia |
| ADR-004 | docs/adrs/ADR-004-garantia-at-least-once.md | Decisão | At-least-once + X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-004-ALT | docs/adrs/ADR-004-garantia-at-least-once.md | Trade-off | Exactly-once descartado por complexidade | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-polling-processo-separado.md | Decisão | Processo separado + polling 2s + single-worker | TRANSCRICAO | [09:10] Larissa |
| ADR-005-ALT | docs/adrs/ADR-005-worker-polling-processo-separado.md | Trade-off | Trigger/NOTIFY improvisado descartado | TRANSCRICAO | [09:09] Diego |
| ADR-005-ENTRY | docs/adrs/ADR-005-worker-polling-processo-separado.md | Decisão | Entry a criar src/worker.ts e npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Decisão | Módulo src/modules/webhooks no padrão existente | TRANSCRICAO | [09:27] Bruno |
| ADR-006-CODE-01 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | Referência AppError e error middleware | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CODE-02 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | Referência requireRole / authenticate | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-CODE-03 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | Extensão prevista em order.service changeStatus | CODIGO | src/modules/orders/order.service.ts |
| ADR-006-CODE-04 | docs/adrs/ADR-006-reuso-padroes-projeto.md | Integração | Logger Pino existente | CODIGO | src/shared/logger/index.ts |

## Checagem rápida de cobertura

| Critério | Resultado |
| --- | --- |
| Linhas com Fonte = `TRANSCRICAO` e timestamp `[hh:mm] Nome` | ≈83% das linhas (≥70% exigido) |
| Linhas com Fonte = `CODIGO` e caminho real | 15 linhas (≥5 exigido; FDD-INT-* e ADR-006-CODE-*) |
| Itens identificáveis nos docs com linha no tracker | ≥80% (FRs/NFRs/OOS/RFC/FDD/ADRs mapeados; IDs alinhados aos docs) |
| Itens inventados sem origem | nenhum — ausência de Localização seria motivo de remoção |

Atualize este arquivo sempre que um requisito/decisão novo for adicionado aos docs.

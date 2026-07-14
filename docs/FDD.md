# FDD — Feature Design Document: Webhooks de Notificação de Pedidos

Documento de implementação acionável. Decisões de arquitetura: [RFC](./RFC.md) e [ADRs](./adrs/README.md). Origem: `TRANSCRICAO.md` + código do OMS.

## 1. Contexto e motivação técnica

O OMS (`order-management-api`) gerencia pedidos com máquina de estados e transação atômica em `changeStatus`, mas **não notifica** sistemas externos. Clientes B2B fazem polling em `GET /orders`. Esta feature introduz webhooks outbound desacoplados via outbox, sem colocar HTTP na path crítica de pedidos.

## 2. Objetivos técnicos

- Registrar eventos de mudança de status na **mesma transação** SQL da atualização do pedido.
- Despachar HTTP de forma assíncrona com **worker em processo separado**.
- Expor CRUD de configuração, histórico de deliveries e replay de DLQ.
- Autenticar entregas com **HMAC-SHA256** e secrets rotacionáveis.
- Reutilizar padrões do repositório (`AppError`, Pino, Zod, módulos, `requireRole`).
- Manter latência de entrega tipicamente **&lt; 10s** (poll 2s + processamento).

## 3. Escopo e exclusões

**Incluído**

- Modelo outbox + DLQ no MySQL; worker polling; CRUD webhooks; deliveries; replay admin; HMAC; filtro por status na inserção; snapshot de payload.

**Excluído / adiado** (não implementar nesta fase)

- Notificação por e-mail quando webhook falha repetidamente ([09:37] Larissa).
- Dashboard visual / painel frontend ([09:39–09:40] Larissa).
- Rate limiting de saída ([09:38–09:39] — observar).
- Arquivamento de linhas entregues após 30 dias ([09:08] Diego).
- Multi-worker / ordering global ([09:12–09:13]).

## 4. Fluxos detalhados

### 4.1 Criação do evento na outbox

```
Cliente API → PATCH /orders/:id/status
  → OrderService.changeStatus
    → BEGIN TX
      → valida transição / estoque
      → UPDATE orders + INSERT order_status_history
      → publishWebhookEvent(tx, order, from, to)
           → lista webhooks ativos do customerId
              cujo eventsFilter contém `to`
           → se vazio: no-op
           → senão: para cada webhook, INSERT webhook_outbox
              (id UUID, eventId UUID, webhookId, payload snapshot JSON,
               status=PENDING, attempts=0, createdAt)
    → COMMIT (ou ROLLBACK se qualquer passo falhar)
```

Regras:

- Payload **já renderizado** na inserção (snapshot); não re-ler order no envio ([09:51–09:52]).
- Tamanho do payload JSON ≤ **64KB**; se exceder, falha com erro `WEBHOOK_*` apropriado **antes** do commit (não truncar).
- Status da outbox: `PENDING` | `PROCESSING` | `FAILED` (aguardando retry) | `DELIVERED` (e/ou equivalente acordado na migration).

### 4.2 Processamento pelo worker

```
loop a cada 2s:
  SELECT … FROM webhook_outbox
    WHERE status IN (PENDING, FAILED)
      AND (next_attempt_at IS NULL OR next_attempt_at <= now())
    ORDER BY created_at ASC
    LIMIT batch
  para cada evento:
    marca PROCESSING
    POST url do webhook (timeout 10s)
      headers: Content-Type, X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id
    se 2xx → DELIVERED + registra delivery sucesso
    senão → incrementa attempts, calcula next_attempt_at, status FAILED
            ou move para DLQ se attempts >= 5
```

Entry: `src/worker.ts` → `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`). `PrismaClient` **novo** no processo do worker.

### 4.3 Retry

| Tentativa após falha | Delay até próxima |
| --- | --- |
| 1ª | 1 minuto |
| 2ª | 5 minutos |
| 3ª | 30 minutos |
| 4ª | 2 horas |
| 5ª | 12 horas |

Após a 5ª falha permanente → DLQ. Timeout HTTP (10s) conta como falha. Resposta não-2xx conta como falha.

### 4.4 DLQ e replay

1. Mover registro para `webhook_dead_letter` (payload, motivo, timestamps, referências webhook/order/event).
2. Remover ou marcar de forma a não ser repescado pela outbox ativa.
3. `POST /admin/webhooks/dead-letter/:id/replay` (ADMIN): recola na outbox como `PENDING`, `attempts=0`; **logar** `userId` do admin que executou ([09:36] Sofia).

## 5. Contratos públicos

Base path sugerido: `/api/webhooks` (ajustável ao prefixo já usado em `buildApiRouter`). Todos os endpoints (exceto o POST outbound do worker) exigem `Authorization: Bearer <JWT>`, salvo onde indicado.

### 5.1 `POST /webhooks` — criar configuração

**Request**

```http
POST /webhooks HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

{
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "url": "https://atlas.example.com/hooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `201`**

```json
{
  "data": {
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "customerId": "550e8400-e29b-41d4-a716-446655440000",
    "url": "https://atlas.example.com/hooks/orders",
    "events": ["SHIPPED", "DELIVERED"],
    "active": true,
    "secret": "whsec_8f3a…",
    "createdAt": "2026-07-14T12:00:00.000Z",
    "updatedAt": "2026-07-14T12:00:00.000Z"
  }
}
```

`secret` é gerada pelo servidor e retornada **somente** na criação (e na rotação). Autenticação: qualquer role autenticada.

**Erros:** `400` `WEBHOOK_INVALID_URL` (não HTTPS); `400` `WEBHOOK_SECRET_REQUIRED` / validação; `404` customer inexistente.

---

### 5.2 `GET /webhooks?customerId=` — listar

**Request**

```http
GET /webhooks?customerId=550e8400-e29b-41d4-a716-446655440000&page=1&pageSize=20 HTTP/1.1
Authorization: Bearer <token>
```

**Response `200`**

```json
{
  "data": [
    {
      "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "customerId": "550e8400-e29b-41d4-a716-446655440000",
      "url": "https://atlas.example.com/hooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-14T12:00:00.000Z",
      "updatedAt": "2026-07-14T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Secret **não** é listada. Paginação no padrão de `src/shared/http/response.ts`.

---

### 5.3 `PATCH /webhooks/:id` — editar

**Request**

```http
PATCH /webhooks/7c9e6679-7425-40de-944b-e07fc1f90ae7 HTTP/1.1
Authorization: Bearer <token>
Content-Type: application/json

{
  "url": "https://atlas.example.com/hooks/orders/v2",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

**Response `200`:** objeto `data` atualizado (sem secret).

**Erros:** `404` `WEBHOOK_NOT_FOUND`; `400` `WEBHOOK_INVALID_URL`.

---

### 5.4 `DELETE /webhooks/:id` — remover

**Request**

```http
DELETE /webhooks/7c9e6679-7425-40de-944b-e07fc1f90ae7 HTTP/1.1
Authorization: Bearer <token>
```

**Response `204`:** sem body.

**Erros:** `404` `WEBHOOK_NOT_FOUND`.

---

### 5.5 `GET /webhooks/:id/deliveries` — histórico de entregas

**Request**

```http
GET /webhooks/7c9e6679-7425-40de-944b-e07fc1f90ae7/deliveries?page=1&pageSize=100 HTTP/1.1
Authorization: Bearer <token>
```

**Response `200`**

```json
{
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "eventId": "d290f1ee-6c54-4b01-90e6-d701748f0851",
      "success": true,
      "httpStatus": 200,
      "responseBody": "{\"ok\":true}",
      "durationMs": 142,
      "payload": {
        "event_id": "d290f1ee-6c54-4b01-90e6-d701748f0851",
        "event_type": "order.status_changed",
        "timestamp": "2026-07-14T12:01:00.000Z",
        "order_id": "…",
        "order_number": "ORD-000042",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "…",
        "total_cents": 15990
      },
      "createdAt": "2026-07-14T12:01:00.200Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Contrato pedido: últimos entregues com sucesso/falha, payload, response, tempo ([09:34] Marcos). Default sugerido: até 100 por página.

---

### 5.6 `POST /webhooks/:id/rotate-secret` — rotação de secret

**Request**

```http
POST /webhooks/7c9e6679-7425-40de-944b-e07fc1f90ae7/rotate-secret HTTP/1.1
Authorization: Bearer <token>
```

**Response `200`**

```json
{
  "data": {
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "secret": "whsec_new…",
    "previousSecretExpiresAt": "2026-07-15T12:00:00.000Z"
  }
}
```

Grace period: secret anterior válida por **24h** para verificação HMAC ([09:21] Sofia).

---

### 5.7 `POST /admin/webhooks/dead-letter/:id/replay` — replay DLQ

**Request**

```http
POST /admin/webhooks/dead-letter/9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d/replay HTTP/1.1
Authorization: Bearer <token-admin>
```

**Response `200`**

```json
{
  "data": {
    "deadLetterId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "outboxId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "status": "PENDING"
  }
}
```

**Auth:** `authenticate` + `requireRole('ADMIN')`.  
**Erros:** `403` `FORBIDDEN`; `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

---

### 5.8 Payload e headers do POST outbound (worker → cliente)

**Headers**

| Header | Conteúdo |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (outbox) |
| `X-Signature` | HMAC-SHA256 do body com secret (e, no grace, aceitar secret antiga no cliente) |
| `X-Timestamp` | timestamp ISO do envio |
| `X-Webhook-Id` | id do endpoint cadastrado |

**Body (exemplo)**

```json
{
  "event_id": "d290f1ee-6c54-4b01-90e6-d701748f0851",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-14T12:01:00.000Z",
  "order_id": "c56a4180-65aa-42ec-a945-5fd21dec0538",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "550e8400-e29b-41d4-a716-446655440000",
  "total_cents": 15990
}
```

Sem `items` ([09:43] Diego). Cliente deduplica por `event_id` / `X-Event-Id`.

## 6. Matriz de erros (`WEBHOOK_*`)

Formato de resposta alinhado a `error.middleware.ts`:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "…", "details": {} } }
```

| Código | HTTP | Quando |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Webhook id inexistente |
| `WEBHOOK_INVALID_URL` | 400 | URL não HTTPS / inválida |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Operação que exige secret e estado inconsistente |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Snapshot &gt; 64KB |
| `WEBHOOK_INACTIVE` | 409 | Operação incompatível com webhook inativo (se aplicável) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | DLQ id inexistente |
| `WEBHOOK_DELIVERY_FAILED` | — | Uso interno / registro em delivery (não necessariamente HTTP da API) |
| `VALIDATION_ERROR` | 400 | Zod (mantido pelo middleware existente) |
| `FORBIDDEN` | 403 | Replay sem role ADMIN |
| `UNAUTHORIZED` | 401 | JWT ausente/inválido |

Erros de domínio estendem `AppError` / subclasses em `src/shared/errors/` ([09:28–09:29] Bruno).

## 7. Estratégias de resiliência

| Mecanismo | Parâmetro |
| --- | --- |
| Timeout HTTP | 10s |
| Retry / backoff | 5× — 1m / 5m / 30m / 2h / 12h |
| DLQ | Tabela `webhook_dead_letter` |
| Fallback | Replay manual admin (sem auto-revive) |
| Entrega | At-least-once; dedup no cliente via `X-Event-Id` |
| Isolamento | Worker ≠ processo API |

## 8. Observabilidade

### Logs (Pino)

Reutilizar `src/shared/logger/index.ts`. Eventos sugeridos (sem novo framework):

- `webhook_outbox_enqueued` — `{ eventId, orderId, webhookId, toStatus }`
- `webhook_delivery_attempt` — `{ eventId, attempt, webhookId }`
- `webhook_delivery_result` — `{ eventId, success, httpStatus, durationMs }`
- `webhook_moved_to_dlq` — `{ eventId, reason }`
- `webhook_dlq_replayed` — `{ deadLetterId, adminUserId }` (**obrigatório** para auditoria)

Redaction já cobre secrets em paths configurados; **nunca** logar secret em claro.

### Métricas

Derivadas do estado das tabelas e do worker (expostas via logs agregáveis ou contadores internos — a reunião não fixou stack de métricas):

- `webhook_outbox_pending_count`
- `webhook_delivery_success_total` / `webhook_delivery_failure_total`
- `webhook_dlq_depth`
- `webhook_delivery_duration_ms` (também persistido em deliveries)
- latência enqueue→primeira tentativa (para validar SLO &lt;10s)

### Tracing / correlação

- Propagar `eventId` (`X-Event-Id`) em todos os logs do ciclo de vida do evento.
- Na API HTTP, manter uso do `requestId` já presente no error middleware (`req.id`).
- Relacionar replay admin: `requestId` + `adminUserId` + `deadLetterId` + novo `outboxId`/`eventId`.

Não se introduz novo produto de APM nesta fase além do correlacionamento via ids já decididos/existentes.

## 9. Dependências e compatibilidade

- Node ≥ 20, Express, Prisma/MySQL, Zod, Pino, JWT — stack atual (`package.json`).
- Novo script `npm run worker` (previsto na call; ainda não existe no `package.json` — a adicionar na implementação).
- Migrations Prisma para tabelas de configuração, outbox, deliveries, dead letter.
- Compatível com enum `OrderStatus` existente em `prisma/schema.prisma`.

## 10. Integração com o sistema existente

| Arquivo existente | Integração |
| --- | --- |
| `src/modules/orders/order.service.ts` | Em `changeStatus`, após update/history (e estoque), chamar `publishWebhookEvent(tx, order, from, to)` **dentro** da mesma `$transaction`. Falha na outbox ⇒ rollback do status. |
| `src/modules/orders/order.status.ts` | Fonte da verdade dos status/`canTransition`; filtro de eventos do webhook usa os mesmos valores do enum `OrderStatus`. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Novas classes/códigos `WEBHOOK_*` seguindo `AppError` / subclasses (`NotFoundError`, `ValidationError`, `ConflictError`, etc.). |
| `src/middlewares/error.middleware.ts` | Sem mudança de contrato: erros `AppError` e Zod já serializados. |
| `src/middlewares/auth.middleware.ts` | `authenticate` no CRUD; `requireRole('ADMIN')` no replay DLQ (mesmo padrão de `user.routes.ts`). |
| `src/shared/logger/index.ts` | Logger único do worker e da API. |
| `src/server.ts` | Referência de entry point; criar análogo `src/worker.ts` sem HTTP listen. |
| `src/routes/index.ts` | Registrar `buildWebhookRouter` (e rota admin) no `buildApiRouter`. |
| `prisma/schema.prisma` | Novos models + FKs para `Customer` / UUIDs no padrão atual. |

Função de fronteira preferida (call):

```ts
publishWebhookEvent(tx, order, fromStatus, toStatus): Promise<void>
```

Não é obrigatório injetar o repository inteiro no `OrderService` ([09:41]).

## 11. Critérios de aceite técnicos

- [ ] Mudança de status e insert outbox são atômicos (teste de rollback).
- [ ] Worker processa em ≤ ~2s + tempo HTTP; requisito de negócio &lt;10s atendido em ambiente saudável.
- [ ] Retry respeita os 5 intervalos; 6º destino é DLQ.
- [ ] Replay admin exige ADMIN e gera log de auditoria.
- [ ] URL `http://` rejeitada; HTTPS aceita.
- [ ] Assinatura HMAC verificável; rotação com grace 24h.
- [ ] Payload sem items; headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` presentes.
- [ ] Filtro de eventos na inserção (não gera outbox se nenhum webhook escuta o status).
- [ ] `GET …/deliveries` expõe sucesso/falha, payload, response, duração.
- [ ] Códigos de erro de domínio usam prefixo `WEBHOOK_`.
- [ ] Review de segurança (Sofia) agendada antes do deploy.

## 12. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- |
| Worker fora do ar | Média | Alto — outbox acumula | Healthcheck/process manager; alertar `pending_count` |
| Cliente sem dedup | Média | Médio — efeitos duplicados | Documentar `X-Event-Id` no portal (Marcos) |
| Carga no MySQL pelo polling | Baixa | Médio | Índice status+created_at; batch pequeno; single worker |
| Secret vazada no cliente | Baixa | Alto | Secret por endpoint + rotação 24h |
| Payload &gt; 64KB | Baixa | Médio | Erro explícito; payload enxuto sem items |

## Referências

- [RFC](./RFC.md)
- [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) … [ADR-006](./adrs/ADR-006-reuso-padroes-projeto.md)
- `TRANSCRICAO.md`

# ADR-006: Reuso dos padrões existentes do projeto

## Status

Aceito

- **Data:** derivado da reunião técnica documentada em `TRANSCRICAO.md`
- **Decisores:** Bruno (Pedidos), Larissa (Tech Lead), Diego (Plataforma)

## Contexto

O OMS já possui convenções claras de módulos, erros, validação, autenticação/autorização e logging. Introduzir um módulo de webhooks com stack ou padrões paralelos aumentaria custo cognitivo e risco de tratamento inconsistente de erros/auditoria.

## Decisão

Implementar webhooks como módulo alinhado ao restante do código:

| Padrão existente | Como o módulo de webhooks reutiliza |
| --- | --- |
| Módulo por domínio (`controller` / `service` / `repository` / `routes` / `schemas`) | Criar `src/modules/webhooks/` no mesmo formato dos demais módulos. |
| Hierarquia de erros (`AppError` e subclasses) | Expor erros de domínio webhook via `AppError` / subclasses em `src/shared/errors/`; prefixo de código **`WEBHOOK_*`**. |
| Middleware de erro centralizado | Não alterar o contrato: `src/middlewares/error.middleware.ts` já serializa `AppError`, Zod e Prisma. |
| Logger Pino | Reusar `src/shared/logger/index.ts`; sem framework de log novo. |
| Auth JWT + `requireRole` | CRUD autenticado com `authenticate`; replay de DLQ com `requireRole('ADMIN')` em `src/middlewares/auth.middleware.ts`. |
| Validação Zod | Schemas no módulo (`webhook.schemas.ts`), inclusive rejeição de URL não-HTTPS. |
| Transação em `changeStatus` | Estender `src/modules/orders/order.service.ts` para chamar `publishWebhookEvent(tx, …)` **dentro** da mesma `$transaction`. |
| IDs UUID | Outbox e entidades novas usam UUID, alinhado a `prisma/schema.prisma`. |

Lógica de processamento do worker vive no módulo (ex.: arquivos a criar `webhook.worker.ts` / `webhook.processor.ts`), com entry point separado a criar `src/worker.ts`.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
| --- | --- |
| Stack/padrões paralelos só para webhooks (outro logger, outro formato de erro, microserviço) | Divergência desnecessária; time pequeno; middleware e erros existentes já cobrem o caso. ([09:27–09:30] Bruno, Larissa, Diego) |
| Injetar `WebhookRepository` inteiro no `OrderService` | Preferida função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebendo o client da transação. ([09:41] Bruno, Diego) |

## Consequências

**Positivas**

- Onboarding e review mais rápidos; mesmo “look and feel” do restante do OMS.
- Erros e auth passam pelos middlewares já testados.
- Integração atômica com pedidos sem acoplamento estrutural excessivo (função + `tx`).

**Negativas / trade-offs**

- `OrderService` ganha uma dependência de publicação de evento (mitigada pela função de fronteira explícita).
- Prefixo `WEBHOOK_*` deve ser mantido com disciplina para não misturar códigos genéricos (`NOT_FOUND`) onde o produto esperar código específico.

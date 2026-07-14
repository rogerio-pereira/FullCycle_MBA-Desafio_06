# Architectural Decision Records

Registros das decisões arquiteturais da feature **Sistema de Webhooks de Notificação de Pedidos**, derivadas da reunião em `TRANSCRICAO.md` e alinhadas ao código existente do OMS.

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](./ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL | Aceito |
| [ADR-002](./ADR-002-retry-backoff-dlq.md) | Política de retry com backoff e DLQ | Aceito |
| [ADR-003](./ADR-003-autenticacao-hmac-sha256.md) | Autenticação HMAC-SHA256 com secret por endpoint | Aceito |
| [ADR-004](./ADR-004-garantia-at-least-once.md) | Garantia at-least-once com `X-Event-Id` | Aceito |
| [ADR-005](./ADR-005-worker-polling-processo-separado.md) | Worker em processo separado com polling | Aceito |
| [ADR-006](./ADR-006-reuso-padroes-projeto.md) | Reuso dos padrões existentes do projeto | Aceito |

Formato: MADR simplificado (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).

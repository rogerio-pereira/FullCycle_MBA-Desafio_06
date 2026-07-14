# ADR-002: Política de retry com backoff e DLQ

## Status

Aceito

- **Data:** derivado da reunião técnica documentada em `TRANSCRICAO.md`
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos), Marcos (PM)

## Contexto

Endpoints de clientes B2B podem ficar indisponíveis (falha transitória, manutenção planejada de horas). Sem política clara, eventos ficam pendurados indefinidamente ou são descartados cedo demais. Também é necessário um destino auditável para falhas permanentes e um caminho operacional de reprocessamento.

## Decisão

1. **Retry com backoff exponencial**, teto de **5 tentativas**, progressão: **1m → 5m → 30m → 2h → 12h** (janela da ordem de ~15h entre primeira falha e última tentativa).
2. Após esgotar tentativas, mover para **DLQ** em tabela separada `webhook_dead_letter` (payload, motivo da falha, timestamp), mantendo a outbox principal limpa para leitura operacional.
3. Reprocessamento **manual** via endpoint admin: `POST /admin/webhooks/dead-letter/:id/replay`, recolocando o evento na outbox como pendente, com **role ADMIN** e auditoria de quem executou o replay.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
| --- | --- |
| Retry indefinido com backoff | Evento fica pendurado eternamente se o cliente desaparecer. ([09:15] Diego) |
| Apenas 3 tentativas (janela curta) | Insuficiente para manutenções planejada de ~2h já observadas em clientes. ([09:16] Bruno, Diego) |
| Marcar `failed` na própria outbox (sem tabela DLQ) | Mistura fila operacional com evidência de falha permanente; DLQ separada facilita debug e replay. ([09:17–09:18] Larissa, Diego) |

## Consequências

**Positivas**

- Cobre janelas de indisponibilidade realistas sem retenção infinita.
- DLQ separada melhora operação, evidência e reprocessamento controlado.
- Replay admin explícito evita “auto-revive” silencioso de eventos mortos.

**Negativas / trade-offs**

- Até ~15h de atraso na última tentativa — aceitável para o PM se o cliente está fora por tanto tempo.
- Exige disciplina operacional (monitorar DLQ; quem pode dar replay).
- Cliente pode receber evento muito após a mudança de status (ainda at-least-once).

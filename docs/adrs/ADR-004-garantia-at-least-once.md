# ADR-004: Garantia at-least-once com `X-Event-Id`

## Status

Aceito

- **Data:** derivado da reunião técnica documentada em `TRANSCRICAO.md`
- **Decisores:** Diego (Plataforma), Sofia (Segurança), Larissa (Tech Lead), Marcos (PM)

## Contexto

Com outbox + worker + retries, o mesmo evento pode ser entregue mais de uma vez (timeout após o cliente processar, crash entre envio e marcação de entregue, replay de DLQ, etc.). Exactly-once distribuído exigiria coordenação complexa entre emissor e receptor.

## Decisão

1. Garantia de entrega: **at-least-once**.
2. Cada evento na outbox recebe um **UUID** (`event_id`) na inserção.
3. Em cada tentativa de entrega, enviar header **`X-Event-Id`** com esse UUID.
4. Documentar no portal de desenvolvedor que o cliente deve **deduplicar** pelo `event_id`.

Headers adicionais alinhados à entrega (definidos na mesma discussão): `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
| --- | --- |
| Exactly-once | Exigiria coordenação nos dois lados; complexidade desproporcional; at-least-once + id cobre o caso típico (referência a práticas Stripe/GitHub). ([09:25] Diego, Sofia) |

## Consequências

**Positivas**

- Modelo operacionalmente simples e alinhado a webhooks de mercado.
- Dedup claro e barato no receptor via `X-Event-Id`.
- Compatível com retries, timeouts e replay de DLQ.

**Negativas / trade-offs**

- Responsabilidade de idempotência fica no cliente (mitigada por documentação no portal).
- Clientes mal preparados podem processar efeitos colaterais duplicados.

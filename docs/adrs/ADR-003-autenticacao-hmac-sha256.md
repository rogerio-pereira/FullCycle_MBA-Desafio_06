# ADR-003: Autenticação HMAC-SHA256 com secret por endpoint

## Status

Aceito

- **Data:** derivado da reunião técnica documentada em `TRANSCRICAO.md`
- **Decisores:** Sofia (Segurança), Larissa (Tech Lead), Bruno (Pedidos), Diego (Plataforma)

## Contexto

Webhooks outbound expõem dados de pedidos a endpoints fora da nossa infra. O receptor precisa verificar origem e integridade do payload. Já houve caso de secret vazada em log do lado do cliente, o que torna secret global e/ou não rotacionável inaceitável.

## Decisão

1. Assinar o corpo do request com **HMAC-SHA256** e enviar a assinatura no header **`X-Signature`**.
2. Cada endpoint de webhook do cliente possui **secret única** (não há secret global da plataforma).
3. Secret é **gerada pela API** na criação do webhook e é **rotacionável** via endpoint; a secret antiga permanece válida por **24h** (grace period) e depois é invalidada.
4. **TLS obrigatório**: URL cadastrada deve ser `https`; `http` é rejeitado na validação (schema Zod).
5. Limite de payload de envio: **64KB**; se ultrapassar, falhar (não truncar) — tratado como requisito não funcional / validação, não como ADR separado.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
| --- | --- |
| Secret global única da plataforma | Compromisso de uma secret compromete todos os clientes/endpoints. ([09:21] Sofia) |
| Truncar payloads grandes | Mascararia anomalia; preferência por erro explícito se payload exceder teto. ([09:23–09:24] Sofia, Diego) |

## Consequências

**Positivas**

- Padrão de mercado verificável pelos clientes.
- Blast radius limitado por endpoint; rotação com grace period reduz downtime na migração do cliente.
- Validação HTTPS na borda da API reduz superfície de ataque em trânsito.

**Negativas / trade-offs**

- Cliente precisa implementar verificação HMAC e gerenciar rotação no lado dele.
- Grace period de 24h amplia janela em que duas secrets são válidas (aceitável vs. cutting over abrupto).
- Revisão de segurança (HMAC/geração de secret) é gate antes do deploy.

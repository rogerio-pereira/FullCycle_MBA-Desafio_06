# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks outbound de mudança de status de pedidos |
| **Autor (produto)** | Marcos (Product Manager) |
| **Origem** | Reunião técnica em `TRANSCRICAO.md` + contexto do código existente |
| **Status** | Aprovado em reunião; documentação formal para início de implementação |

## 1. Resumo e contexto da feature

Clientes B2B integrados ao OMS precisam saber imediatamente quando o status de um pedido muda (ex.: `PAID` → `SHIPPED`). Hoje descobrem isso fazendo polling em `GET /orders`, o que é lento e caro para eles. Esta feature entrega **notificações outbound via webhook HTTPS**, assinadas, configuráveis por cliente, com histórico de entregas e operação de reprocessamento para a equipe interna.

A solução técnica (outbox, worker, retry, HMAC) está no [RFC](./RFC.md) e no [FDD](./FDD.md); este PRD fixa o **porquê**, o **quê** e as métricas de sucesso.

## 2. Problema e motivação

- **Dor:** Atlas Comercial, MaxDistribuição e Nova Cargo batem periodicamente em `GET /orders` para detectar mudanças.
- **Risco comercial:** Atlas sugeriu que, sem entrega até o fim do trimestre (prazo alinhado a novembro na call), pode migrar para concorrente.
- **Expectativa de “tempo real”:** qualquer latência **abaixo de 10 segundos** já satisfaz o cliente; não precisam de push sub-segundo.

## 3. Público-alvo e cenários de uso

| Público | Cenário |
| --- | --- |
| Clientes B2B (integradores) | Cadastram URL HTTPS, escolhem quais status ouvir, recebem POST assinado, consultam deliveries, rotacionam secret. |
| Operadores autenticados (JWT) | Gerenciam CRUD de webhooks dos customers via API. |
| Administradores (`ADMIN`) | Reprocessam eventos em DLQ com auditoria. |
| Time interno (engenharia/segurança) | Opera worker, revisa HMAC/secrets antes do deploy. |

**Cenário feliz:** pedido muda para `SHIPPED` → evento na outbox → worker entrega em poucos segundos → cliente atualiza WMS/ERP sem polling.

**Cenário de falha transitória:** endpoint do cliente fora do ar → retries com backoff → eventualmente DLQ → admin faz replay após recuperação.

## 4. Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Entregar “tempo real” percebido pelo cliente | Latência p95 entre mudança de status commitada e **primeira tentativa** de entrega HTTP | **&lt; 10 segundos** |
| Reduzir dependência de polling | Clientes-piloto (Atlas, MaxDistribuição, Nova Cargo) deixam de depender de polling agressivo para status | Adoção do webhook pelos **3** clientes solicitantes até o go-live acordado |
| Entrega confiável sob falha | Eventos não entregues após esgotar retries ficam auditáveis em DLQ com caminho de replay | **100%** dos esgotados vão para DLQ (sem drop silencioso) |
| Prazo comercial | Entrega da feature | **3 sprints**, incluindo review de segurança de Sofia |

## 5. Escopo

### 5.1 Incluso

- Webhooks **somente outbound** (OMS → cliente).
- CRUD de configuração (URL, filtro de status, ativo/inativo; secret gerada pelo sistema).
- Histórico de deliveries por webhook.
- Rotação de secret com grace period de 24h.
- Despacho assíncrono com outbox + worker + retry + DLQ + replay admin.
- Autenticação HMAC-SHA256; HTTPS obrigatório.
- Documentação de integração no portal de desenvolvedor (responsabilidade PM).

### 5.2 Fora de escopo

Itens **explicitamente descartados ou adiados** na reunião:

1. **E-mail / alerta** ao cliente quando o webhook falha N vezes seguidas — próxima fase ([09:37–09:38] Marcos, Larissa).
2. **Dashboard visual** / painel frontend para o cliente — projeto separado do time de frontend; apenas API agora ([09:39–09:40] Marcos, Larissa).
3. **Rate limiting** de envio ao endpoint do cliente — observar e decidir depois ([09:38–09:39] Diego, Larissa).
4. **Arquivamento** automático de outbox entregue após 30 dias ([09:08] Diego).
5. **Multi-worker** e garantia de ordering global ([09:12–09:14]).

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| FR-01 | Cliente (via API autenticada) pode **criar** webhook informando URL HTTPS, lista de status desejados e `customerId` (body/path; não derivado do JWT). Secret é **gerada** e devolvida na criação. | [09:31–09:32] Marcos, Larissa |
| FR-02 | Cliente pode **listar**, **editar (PATCH)** e **remover (DELETE)** webhooks de um customer. | [09:33] Bruno, Marcos |
| FR-03 | Por endpoint, filtrar quais status disparam notificação; filtro aplicado na **inserção** da outbox. | [09:33–09:34] Marcos, Bruno, Diego |
| FR-04 | Em toda mudança de status relevante, sistema enfileira evento de forma atômica com a transação de pedidos. | [09:40–09:41] Bruno, Diego |
| FR-05 | Worker entrega POSTs ao URL do cliente com payload JSON de `order.status_changed` (sem items) e headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`. | [09:43–09:45] Diego, Sofia |
| FR-06 | Em falha de entrega, retentar com backoff (5 tentativas: 1m/5m/30m/2h/12h) e, ao esgotar, persistir em **DLQ**. | [09:15–09:18] Diego, Larissa |
| FR-07 | Admin pode **reprocessar** item de DLQ via `POST /admin/webhooks/dead-letter/:id/replay` (role `ADMIN`), com log de quem executou. | [09:18], [09:35–09:36] Diego, Sofia |
| FR-08 | Cliente consulta **histórico de deliveries** (`GET /webhooks/:id/deliveries`): sucesso/falha, payload, response, tempo (até ~100 recentes). | [09:34] Marcos |
| FR-09 | Cliente pode **rotacionar secret**; secret anterior válida por 24h. | [09:21–09:22] Sofia |
| FR-10 | Entrega é **at-least-once**; cliente deduplica por `X-Event-Id` (UUID gerado na outbox). | [09:24–09:26] Diego, Marcos |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| NFR-01 | Latência percebida &lt; 10s (poll do worker a cada 2s). | [09:02], [09:09–09:10] |
| NFR-02 | Timeout HTTP do worker: 10s. | [09:42] Diego |
| NFR-03 | Payload máximo 64KB; erro se ultrapassar (não truncar). | [09:23–09:24] Sofia, Diego, Larissa |
| NFR-04 | TLS obrigatório (somente HTTPS). | [09:23] Sofia |
| NFR-05 | HMAC-SHA256; secret por endpoint (não global). | [09:20–09:21] Sofia |
| NFR-06 | Reuso de padrões do OMS (módulo, AppError, Pino, error middleware, códigos `WEBHOOK_*`). | [09:27–09:30] Bruno, Larissa |
| NFR-07 | Worker em processo separado da API. | [09:11] Diego, Larissa |
| NFR-08 | Review de segurança (≥2 dias úteis) antes do deploy. | [09:46] Sofia |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off | Doc |
| --- | --- | --- |
| Outbox no MySQL vs fila externa / sync HTTP | Simplicidade operacional × carga no mesmo DB; evita acoplar HTTP à TX de pedidos | [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) |
| 5 retries ~15h vs 3 curtos ou infinito | Cobre manutenção de horas × atraso longo na última tentativa | [ADR-002](./adrs/ADR-002-retry-backoff-dlq.md) |
| At-least-once + dedup no cliente vs exactly-once | Simplicidade × responsabilidade no cliente | [ADR-004](./adrs/ADR-004-garantia-at-least-once.md) |
| Single-worker ordering por pedido | Ordenação ok agora × escala horizontal adiada | [ADR-005](./adrs/ADR-005-worker-polling-processo-separado.md) |

## 9. Dependências

- OMS existente (pedidos, JWT, roles `ADMIN`/`OPERATOR`, Prisma/MySQL).
- Disponibilidade de URL HTTPS estável no lado dos clientes B2B.
- Portal de desenvolvedor para documentar integração, HMAC e dedup (Marcos).
- Capacidade do time: ~3 sprints + janela de review de Sofia.
- Processo worker em ambiente de deploy (além da API).

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Cliente B2B não implementa dedup e processa eventos duplicados | Média | Alto (efeito colateral no ERP/WMS) | Destacar at-least-once + `X-Event-Id` no portal; exemplos de verificação HMAC |
| Prazo Atlas (fim de novembro / trimestre) não caber em 3 sprints se houver bloqueio | Média | Alto (risco de churn) | Escopo fechado (sem e-mail/dashboard/rate limit); review de segurança reservada no fim |
| Worker ou outbox mal operados geram atraso &gt;10s ou acúmulo | Média | Médio | Monitorar profundidade da outbox/DLQ; script `npm run worker` como dependência de runtime |
| Vazamento de secret no lado do cliente | Baixa | Alto | Secret por endpoint + rotação com grace 24h (já ocorreram vazamentos em log de cliente) |

## 11. Critérios de aceitação

- [ ] Os 3 clientes solicitantes conseguem cadastrar webhook HTTPS e receber eventos de status escolhidos.
- [ ] Mudança de status sem webhook interessado **não** gera lixo na outbox.
- [ ] Falha do endpoint do cliente não reverte mudança de status no OMS.
- [ ] Após 5 falhas, evento aparece na DLQ e pode ser reprocessado por ADMIN.
- [ ] Deliveries consultáveis com resultado e tempo de resposta.
- [ ] URL HTTP é rejeitada; assinatura HMAC presente em todo envio.
- [ ] Documentação no portal cobre payload, headers e deduplicação.
- [ ] Meta de latência &lt;10s validada em ambiente de homologação com worker saudável.

## 12. Estratégia de testes e validação

| Camada | Foco |
| --- | --- |
| Unitário | Filtro de eventos; cálculo de backoff; validação HTTPS; montagem/assinatura HMAC; snapshot de payload |
| Integração | `changeStatus` + insert outbox na mesma TX (commit/rollback); CRUD; rotate-secret grace; replay ADMIN |
| Contrato | Payloads/headers conforme FDD; códigos `WEBHOOK_*` |
| E2E / homologação | Worker real contra endpoint de teste dos clientes-piloto; medir latência enqueue→entrega; simular 5xx e DLQ |
| Segurança | Review Sofia (HMAC, geração/rotação de secret, redaction de logs) antes do deploy |
| Aceite de produto | Marcos valida com Atlas/MaxDistribuição/Nova Cargo no portal + smoke de entrega |

## Referências

- [RFC](./RFC.md) · [FDD](./FDD.md) · [ADRs](./adrs/README.md) · [TRACKER](./TRACKER.md)
- `TRANSCRICAO.md`

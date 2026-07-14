# ADR-001: Padrão Outbox no MySQL

- **Status:** Aceito
- **Data:** derivado da reunião técnica documentada em `TRANSCRICAO.md`
- **Decisores:** Larissa (Tech Lead), Diego (Plataforma), Bruno (Pedidos)

## Contexto

A mudança de status de pedido já ocorre em uma transação SQL que atualiza `orders`, grava `order_status_history` e, quando aplicável, altera estoque. Clientes B2B precisam ser notificados quando o status muda. Inserir uma chamada HTTP síncrona dentro dessa transação acoplaria a disponibilidade do endpoint do cliente à disponibilidade da operação de pedidos: cliente lento travaria outras mudanças de status, e falha de rede forçaria decidir entre rollback do status (incorreto) ou inconsistência status/notificação.

Também foi avaliada a introdução de infraestrutura de mensageria (ex.: Redis Streams) apenas para essa feature.

## Decisão

Adotar o padrão **Transactional Outbox** no MySQL já usado pelo OMS:

1. Na mesma transação de `changeStatus`, inserir uma linha em tabela `webhook_outbox` com o evento (snapshot do payload).
2. Um worker separado lê a outbox e dispara as chamadas HTTP.
3. Commit da transação principal implica evento registrado; rollback remove o evento junto — sem inconsistência status/outbox.

Arquivamento de linhas já entregues (ex.: após 30 dias) fica fora do escopo desta feature.

## Alternativas Consideradas

| Alternativa | Motivo do descarte |
| --- | --- |
| Chamada HTTP síncrona dentro de `OrderService.changeStatus` | Aumenta latência/contensão da transação; cliente offline levanta dilema incorreto de rollback do status. ([09:03–09:04] Larissa, Bruno) |
| Redis Streams (ou fila dedicada) | Exigiria nova infra (ex.: Redis Cluster); time pequeno; overengineering para o problema atual. ([09:07] Larissa, Diego) |

## Consequências

**Positivas**

- Atomicidade entre mudança de status e registro do evento de notificação.
- Reuso do MySQL/Prisma existentes; sem novo componente operacional na fase 1.
- Worker desacoplado pode falhar/retentar sem afetar o caminho crítico da API de pedidos.

**Negativas / trade-offs**

- A outbox compete pelo mesmo banco; exige índices em status/`created_at` e batches pequenos no worker.
- Delivery deixa de ser imediata: depende do ciclo do worker (polling).
- Operações futuras (arquivamento, particionamento) ficam como débito explícito fora do escopo atual.

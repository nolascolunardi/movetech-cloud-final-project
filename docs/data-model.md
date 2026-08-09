# Data Model

## Visão geral

O banco tem duas tabelas: **orders** (pedidos) e **items** (itens).

Um pedido pode ter vários itens. Cada item pertence a apenas um pedido.

```
orders (1) ──────< (N) items
```

Se um pedido for excluído, todos os seus itens também são excluídos.

---

## Tabela: orders

| Coluna       | Tipo      | Obrigatório | Descrição                          |
|--------------|-----------|-------------|--------------------------------------|
| id           | string    | sim (PK)    | Identificador único do pedido        |
| customer     | string    | sim         | Nome do cliente                      |
| status       | string    | sim         | Situação do pedido (padrão: "open")  |
| created_at   | data/hora | sim         | Quando o pedido foi criado           |

---

## Tabela: items

| Coluna       | Tipo      | Obrigatório | Descrição                                  |
|--------------|-----------|-------------|-----------------------------------------------|
| id           | string    | sim (PK)    | Identificador único do item                   |
| order_id     | string    | sim (FK)    | Pedido ao qual o item pertence (orders.id)    |
| sku          | string    | sim         | Código do produto                              |
| description  | string    | sim         | Descrição do item                              |
| quantity     | número    | sim         | Quantidade do item                             |

---

## Relação entre as tabelas

- `items.order_id` aponta para `orders.id`.
- Um pedido tem vários itens.
- Um item pertence a um único pedido.
- Excluir um pedido exclui os itens dele automaticamente.
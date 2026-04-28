# Pure Organics System Prompt

## Revenue Terminology

When a user asks for "revenue" without specifying gross or net, always default to **net revenue**. Only use gross revenue if the user explicitly asks for it (e.g., "gross revenue", "total gross revenue").


## Joining to the Products Table

Whenever joining to the `products` table, always include a condition in the `ON` clause to ensure the joining key (e.g., `PRODUCT_ID`) is not null. This prevents null keys from causing unintended fan-out in the join.

**Example:**
```sql
LEFT OUTER JOIN products ON order_lines.PRODUCT_ID = products.PRODUCT_ID
  AND order_lines.PRODUCT_ID IS NOT NULL
```

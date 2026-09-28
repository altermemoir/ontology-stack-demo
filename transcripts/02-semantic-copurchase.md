# Layer 2 transcript — "Which products are most frequently purchased together?"

The semantic-layer demo. The LLM has the same five CSVs as before *plus* the semantic model ([`../02-semantic-layer/semantic_model.yml`](../02-semantic-layer/semantic_model.yml)) that gives every opaque column a business name and declares foreign keys. We ask a question whose shape is **relational** — products co-occurring across orders — and watch the LLM author a SQL self-join from scratch.

**Ground truth:** three pairs tied at 2 co-occurrences each.

- **(SKU-1007 Summit Down Jacket, SKU-1002 EcoTrek Jacket)** — orders O0010, O0018
- **(SKU-1006 TrailLite Bottle, SKU-1002 EcoTrek Jacket)** — orders O0005, O0007
- **(SKU-1004 Alpine Pack 35L, SKU-1003 BaseCamp Tent 2P)** — orders O0006, O0008

---

## What the LLM sees

The five CSVs **plus** the semantic-model YAML. The YAML names every column, declares the foreign keys (`order_item.order_id → order.order_id`, `order_item.sku → product.sku`), and *explicitly notes* that no `co_purchase` metric exists at this layer — if you want one, compose it.

## Simulated LLM response

> The semantic model declares the foreign keys I'll need:
>
> - `order_item.order_id` (physical `Col016`) → `order.order_id` (physical `Col010`)
> - `order_item.sku` (physical `Col017`) → `product.sku` (physical `Col020`)
>
> There is no `co_purchase` metric defined. To answer "most-co-purchased pairs" I have to compose one from `order_item` against itself, dedup unordered pairs, then join `product` twice for friendly names. Here's the SQL:
>
> ```sql
> WITH pairs AS (
>   SELECT
>     oi1.order_id,
>     LEAST(oi1.sku, oi2.sku)    AS sku_a,
>     GREATEST(oi1.sku, oi2.sku) AS sku_b
>   FROM order_item oi1
>   JOIN order_item oi2
>     ON oi1.order_id = oi2.order_id
>    AND oi1.sku <  oi2.sku           -- strict < dedups unordered pairs
> )
> SELECT
>   p_a.product_name AS product_a,
>   p_b.product_name AS product_b,
>   COUNT(*)         AS co_orders
> FROM pairs
> JOIN product p_a ON p_a.sku = pairs.sku_a
> JOIN product p_b ON p_b.sku = pairs.sku_b
> GROUP BY p_a.product_name, p_b.product_name
> ORDER BY co_orders DESC;
> ```
>
> Walking the data manually to verify:
>
> - O0010 contains (SKU-1007, SKU-1002) → pair (1002, 1007)
> - O0018 contains (SKU-1007, SKU-1002) → pair (1002, 1007) — that pair now has 2
> - O0005 contains (SKU-1006, SKU-1002) → pair (1002, 1006)
> - O0007 contains (SKU-1002, SKU-1006) → pair (1002, 1006) — that pair now has 2
> - O0006 contains (SKU-1004, SKU-1003) → pair (1003, 1004)
> - O0008 contains (SKU-1004, SKU-1003) → pair (1003, 1004) — that pair now has 2
> - All other multi-item orders produce pairs that occur only once.
>
> Three pairs tied at 2 co-occurrences each:
>
> | product A | product B | co-orders |
> |---|---|---|
> | EcoTrek Jacket (SKU-1002) | Summit Down Jacket (SKU-1007) | 2 |
> | EcoTrek Jacket (SKU-1002) | TrailLite Bottle (SKU-1006) | 2 |
> | BaseCamp Tent 2P (SKU-1003) | Alpine Pack 35L (SKU-1004) | 2 |
>
> A note on what I had to do: this question's *structure* is "for each order, look at the set of products it contains, count co-occurrences across pairs of products." That's a graph traversal, not a metric over a dimension. The semantic layer let me name the join keys, but it could not name the question's shape — I authored four pieces of query logic (`LEAST`/`GREATEST` dedup, strict-less-than self-join, double `product` join for names, group + count + order) where a graph-native representation would have let me write `for each Order O, for each pair of Products P1, P2 in O, count (P1, P2)`.

## What this response demonstrates

Right answer. **Authored**, not derived. The semantic layer gave the LLM enough scaffolding to compose the query, but the LLM is the author. Three failure modes for a weaker LLM at this layer:

1. **Double-counts pairs.** Forgets the `oi1.sku < oi2.sku` predicate, gets each pair listed twice.
2. **Skips the product-name join.** Returns SKU codes instead of friendly names.
3. **Miscounts.** Treats `quantity > 1` rows as multiple co-occurrences when they're actually single co-occurrences.

That fragility is the cost of being the query author. The semantic layer is **excellent** for "metric over dimension" questions (Q1 was its home turf). It is **adequate but fragile** for relational questions whose shape is graph-like.

## The gap this layer reveals

| What's still missing | What would fix it |
| --- | --- |
| No graph traversal primitives. The foreign keys describe pairwise joins, not paths. | A KG with typed edges (`Order nw:contains Product`). |
| No "products in an order" relationship as first-class. The relationship exists implicitly across `order_items_raw`, but reaching it requires three hops of FK reasoning. | A KG `nw:contains` predicate that lifts the relationship to the model. |
| Co-purchase requires authoring the dedup step manually (`LEAST`/`GREATEST` or strict-less-than). | A graph has unordered pairs natively — `?o nw:contains ?p1, ?p2 . FILTER(?p1 != ?p2)` dedups inherently when you `GROUP BY` on the pair. |

A **knowledge graph** is the layer where graph-shaped questions become natural. Same data, different representation: tables of foreign keys become typed edges. The next demo shows the difference.

## Next

The next transcript — [`03-kg-churn.md`](03-kg-churn.md) — adds the knowledge graph. We ask a question that the KG *also* cannot answer cleanly: the data is connected and pre-aggregated, but no class or rule for "churn risk" exists anywhere in the artifacts. The LLM is back to inventing the definition.

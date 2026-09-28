# Layer 2 — Lakehouse + Semantic Layer (live demo prompt)

**What this demo shows:** the semantic layer gave us business names and metric definitions — perfect for "what was our revenue last year?". But the moment we ask a question that is *structurally a graph* — "which products are most often bought together?" — the LLM has to author a multi-CTE self-join from scratch. The answer comes out right, but the cost is **the LLM is the query author**, and the join logic is fragile. **This is the case for a knowledge graph.**

Paste these blocks into a **fresh** LLM conversation. Do not reuse the chat from Layer 1 — start a new one to avoid context leakage.

## Step 1 — paste this as your only message

```text
You are a senior data analyst at NorthWind Outdoor, a fictional online outdoor-gear retailer. Today's date is 2026-06-01.

You have been given two artifacts: (a) the same five raw lakehouse CSV extracts from before, with opaque column headers (Col001 - Col031, globally unique across all tables); and (b) a semantic-model YAML that maps every physical column to a business name and defines metrics, dimensions, and foreign keys. Each artifact is delimited by a "=== File: <name> ===" marker. Treat the semantic model as the source of truth for what each column means and how the tables join.

CLOSED-LOOP CONSTRAINT (read this first):
This is a closed-loop evaluation. You may use ONLY the artifacts pasted in this message and your reasoning over them. Nothing else. Specifically, you must NOT:
- browse the web, search, or call any retrieval tool;
- invoke training-corpus knowledge about regulations, geography, industry conventions, or external ontologies you may have seen elsewhere;
- silently use general knowledge to fill a gap. If you find yourself reaching for it, stop and say "this requires X, which the artifacts above do not provide."

If a question requires information not in the artifacts, the correct response is to identify the gap explicitly and stop — not to guess from outside knowledge.

Layer-specific guidance:
- The semantic model defines metrics, dimensions, and foreign keys; treat its definitions as authoritative.
- The semantic model does NOT define a "co-purchase" metric. If you need one, you must compose it yourself from the underlying tables and foreign keys.
- Show your work. If you compose a query, paste it. The shape of the query is the point of this question — not just the final list.

=== File: customers_raw.csv ===
Col001,Col002,Col003,Col004,Col005,Col006,Col007,Col008,Col009
C001,Anna,Schmidt,a.schmidt@example.de,2022-03-15,DE,P,2025-08-30,N
C002,Marco,Rossi,marco.rossi@example.it,2021-07-02,IT,G,2025-11-20,Y
C003,Yuki,Tanaka,yuki.tanaka@example.jp,2020-02-14,JP,P,2024-11-05,N
C004,Sarah,Johnson,sjohnson@example.com,2023-01-10,US,G,2026-04-12,N
C005,Hans,Mueller,h.mueller@example.de,2019-06-01,DE,P,2025-09-15,Y
C006,Emma,Davies,emma.davies@example.co.uk,2024-05-22,GB,S,2026-03-30,Y
C007,Carlos,Garcia,c.garcia@example.es,2022-11-08,ES,B,2025-09-18,N
C008,Priya,Patel,priya.patel@example.co.uk,2021-03-19,GB,G,2025-11-25,N
C009,Klaus,Weber,k.weber@example.de,2018-09-01,DE,P,2024-08-20,N
C010,Sophie,Laurent,s.laurent@example.fr,2023-08-04,FR,S,2026-02-14,Y

=== File: orders_raw.csv ===
Col010,Col011,Col012,Col013,Col014,Col015
O0001,C001,2025-02-14,COMPLETED,600.00,DE
O0002,C001,2025-05-22,COMPLETED,450.00,DE
O0003,C001,2025-08-30,COMPLETED,525.00,DE
O0004,C001,2026-01-15,RETURNED,40.00,DE
O0005,C002,2025-03-10,COMPLETED,415.00,IT
O0006,C002,2025-07-15,COMPLETED,600.00,IT
O0007,C002,2025-11-20,COMPLETED,590.00,IT
O0008,C003,2024-11-05,COMPLETED,698.00,JP
O0009,C004,2025-04-12,COMPLETED,800.00,US
O0010,C004,2025-10-30,COMPLETED,950.00,US
O0011,C004,2026-04-12,COMPLETED,199.00,US
O0012,C005,2025-06-08,COMPLETED,615.00,DE
O0013,C005,2025-09-15,COMPLETED,175.00,DE
O0014,C006,2025-12-21,COMPLETED,160.00,GB
O0015,C006,2026-03-30,COMPLETED,65.00,GB
O0016,C007,2025-09-18,COMPLETED,25.00,ES
O0017,C008,2025-04-25,COMPLETED,800.00,GB
O0018,C008,2025-09-08,COMPLETED,750.00,GB
O0019,C008,2025-11-25,COMPLETED,569.00,GB
O0020,C009,2024-08-20,COMPLETED,349.00,DE
O0021,C010,2026-02-14,COMPLETED,220.00,FR

=== File: order_items_raw.csv ===
Col016,Col017,Col018,Col019
O0001,SKU-1007,1,349.00
O0001,SKU-1004,1,175.00
O0002,SKU-1002,2,219.50
O0003,SKU-1007,1,349.00
O0003,SKU-1005,1,39.95
O0004,SKU-1005,1,39.95
O0005,SKU-1006,1,24.99
O0005,SKU-1002,1,219.50
O0006,SKU-1004,1,175.00
O0006,SKU-1003,1,189.00
O0007,SKU-1002,1,219.50
O0007,SKU-1006,1,24.99
O0008,SKU-1004,1,175.00
O0008,SKU-1003,1,189.00
O0009,SKU-1007,2,349.00
O0010,SKU-1007,1,349.00
O0010,SKU-1002,1,219.50
O0011,SKU-1008,1,199.00
O0012,SKU-1004,1,175.00
O0013,SKU-1006,1,24.99
O0014,SKU-1001,1,159.99
O0015,SKU-1006,2,24.99
O0016,SKU-1006,1,24.99
O0017,SKU-1007,2,349.00
O0018,SKU-1007,1,349.00
O0018,SKU-1002,1,219.50
O0019,SKU-1001,2,159.99
O0019,SKU-1005,1,39.95
O0020,SKU-1007,1,349.00
O0021,SKU-1002,1,219.50

=== File: products_raw.csv ===
Col020,Col021,Col022,Col023,Col024,Col025
SKU-1001,Trail Runner Pro 5,NX,FW,159.99,SYN
SKU-1002,EcoTrek Jacket,EC,AP,219.50,REC
SKU-1003,BaseCamp Tent 2P,BS,EQ,189.00,SYN
SKU-1004,Alpine Pack 35L,NX,EQ,175.00,SYN
SKU-1005,RecyclWool Beanie,EC,AP,39.95,REC
SKU-1006,TrailLite Bottle,BS,EQ,24.99,STL
SKU-1007,Summit Down Jacket,NX,AP,349.00,DWN
SKU-1008,EcoFlex Hiker,EC,FW,199.00,REC

=== File: tickets_raw.csv ===
Col026,Col027,Col028,Col029,Col030,Col031
T001,C003,2025-02-10,OPEN,-0.7,shipping_delay
T002,C009,2025-01-15,OPEN,-0.8,refund_request
T003,C001,2026-01-20,RESOLVED,-0.3,size_exchange
T004,C006,2026-04-02,OPEN,-0.5,defect
T005,C007,2024-12-01,RESOLVED,-0.4,shipping_delay

=== File: semantic_model.yml ===
version: 2

models:
  - name: customer
    description: A registered shopper.
    physical_table: customers_raw
    columns:
      - name: customer_id
        physical_column: Col001
        primary_key: true
      - name: first_name
        physical_column: Col002
      - name: last_name
        physical_column: Col003
      - name: email
        physical_column: Col004
      - name: customer_country
        physical_column: Col006

  - name: order
    description: A single purchase transaction. May be COMPLETED or RETURNED.
    physical_table: orders_raw
    columns:
      - name: order_id
        physical_column: Col010
        primary_key: true
      - name: customer_id
        physical_column: Col011
        foreign_key: customer.customer_id
      - name: order_date
        physical_column: Col012
      - name: order_status
        physical_column: Col013
        codelist:
          COMPLETED: Fulfilled and paid.
          RETURNED: Customer returned the order; revenue reversed.
      - name: order_amount_usd
        physical_column: Col014

  - name: order_item
    description: A single line item within an order.
    physical_table: order_items_raw
    columns:
      - name: order_id
        physical_column: Col016
        foreign_key: order.order_id
      - name: sku
        physical_column: Col017
        foreign_key: product.sku
      - name: quantity
        physical_column: Col018
      - name: unit_price_usd
        physical_column: Col019

  - name: product
    description: A SKU in the catalog.
    physical_table: products_raw
    columns:
      - name: sku
        physical_column: Col020
        primary_key: true
      - name: product_name
        physical_column: Col021
      - name: brand_code
        physical_column: Col022
      - name: category_code
        physical_column: Col023
      - name: list_price_usd
        physical_column: Col024

metrics:
  - name: total_revenue
    description: Sum of completed-order amounts. Excludes RETURNED orders.
    formula: SUM(order.order_amount_usd) WHERE order.order_status = 'COMPLETED'

  - name: customer_lifetime_value
    description: Total completed-order spend grouped by customer.
    formula: SUM(order.order_amount_usd) WHERE order.order_status = 'COMPLETED' GROUP BY order.customer_id

dimensions:
  - name: customer_country
    source: customer.customer_country
  - name: order_year
    source: YEAR(order.order_date)
  - name: product_name
    source: product.product_name

# Notable absences (handled at higher layers):
#   - No "co_purchase" metric. Market-basket / affinity analysis is a multi-hop
#     pattern that the semantic layer cannot express natively. Author it as a
#     self-join if you need it.
#   - No graph-traversal primitives. The foreign keys describe pairwise joins,
#     not paths.

The question: Which products are most frequently purchased together at NorthWind?
```

## What to expect at this layer

The true answer is a 3-way tie at 2 co-occurrences each:

- **(SKU-1007 Summit Down Jacket, SKU-1002 EcoTrek Jacket)** — orders O0010, O0018
- **(SKU-1006 TrailLite Bottle, SKU-1002 EcoTrek Jacket)** — orders O0005, O0007
- **(SKU-1004 Alpine Pack 35L, SKU-1003 BaseCamp Tent 2P)** — orders O0006, O0008

A competent LLM will get there — but **look at the work it has to do** to get there:

1. **Compose a self-join.** `JOIN order_item AS oi1 ON oi1.order_id = oi2.order_id AND oi1.sku < oi2.sku` is not in the semantic model. The LLM authors it.
2. **Dedup unordered pairs.** `LEAST` / `GREATEST` or a strict-less-than predicate, or you double-count (A, B) as separate from (B, A).
3. **Join `product` twice for friendly names.** Once for the left product, once for the right.
4. **Group, count, order.** And explain it.

That's a four-step query authorship task for a question that — at its heart — is just *"for each order, look at the set of products it contains, and count co-occurrences across the pairs."* The LLM produced the SQL. A weaker LLM trips on the dedup, omits a product join, or double-counts. The semantic layer made the joins *expressible* but not *natural*.

## The gap this layer reveals

The semantic layer's strength is **metrics over dimensions**: pure aggregations, named columns, declared filters. That's where Q1 (revenue) was easy.

But the moment the question's shape is **relational** — *which products go together, which customers share neighbors, who bought the same thing as our top spender, what regions cluster on category preference* — the semantic layer is asking the LLM to be a SQL author. The foreign keys are there. The shape of the answer is a graph. The model doesn't speak graph.

A **knowledge graph** is what speaks graph natively. Same data, lifted out of tables and into typed edges. An `Order` has a `contains` edge to each `Product` it includes. Co-purchase becomes "follow `contains` twice." No CTE, no self-join, no `LEAST`/`GREATEST` machinery. The structure of the query collapses down to the structure of the question.

That's what we'll see in the next layer.

Next demo: [03-kg-prompt.md](03-kg-prompt.md) — the same Q4-shaped pattern, but the LLM walks two edges instead of authoring a self-join. And then we'll ask a harder question that even the KG can't answer alone.

## Compare against

- Pre-baked transcript of this layer's response: [02-semantic-copurchase.md](../transcripts/02-semantic-copurchase.md).

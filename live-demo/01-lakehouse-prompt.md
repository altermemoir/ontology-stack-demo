# Layer 1 — Lakehouse only (live demo prompt)

**What this demo shows:** even the simplest business question — "what was our revenue last year?" — turns into a chain of unstated assumptions when all you have is a warehouse with stripped column names. Watch the LLM hedge on what a column means, what a status value implies, and whether to include returns. The answer it lands on may be right, but the path is guesswork. **This is the case for a semantic layer.**

Paste these blocks into a **fresh** LLM conversation (ChatGPT, Claude, Gemini — anything frontier with strong instruction-following). Use a brand-new chat; do not reuse a session from another layer.

## Step 1 — paste this as your only message

```text
You are a senior data analyst at NorthWind Outdoor, a fictional online outdoor-gear retailer. Today's date is 2026-06-01.

The company has handed you the raw lakehouse tables below — five CSV extracts, no documentation, opaque column names (Col001 - Col031, globally unique across all five tables), undocumented codelists. Each CSV is delimited by a "=== File: <name> ===" marker. The first row of each file is the header. There is no schema document and no metadata layer in this message — only the headers and the data.

CLOSED-LOOP CONSTRAINT (read this first):
This is a closed-loop evaluation. You may use ONLY the artifacts pasted in this message and your reasoning over them. Nothing else. Specifically, you must NOT:
- browse the web, search, or call any retrieval tool;
- invoke training-corpus knowledge about regulations (GDPR, UK GDPR, CCPA, ePrivacy, CAN-SPAM, etc.), geography (which countries are in the EU/EEA, etc.), industry conventions ("typical churn rules," "typical VIP thresholds," "what most retailers do"), or external ontologies you may have seen elsewhere;
- silently use general knowledge to fill a gap. If you find yourself reaching for it, stop and say "this requires X, which the artifacts above do not provide."

If a question requires information not in the artifacts, the correct response is to identify the gap explicitly and stop — not to guess from outside knowledge. The point is to evaluate what THIS data stack supports, not what a frontier LLM happens to know.

Layer-specific guidance:
- For concepts the data does NOT define (e.g. "revenue," "return," "completed order"), say so explicitly. If you choose to offer a heuristic guess, label it "my heuristic, not from the artifacts" — do not present it as derived.
- Show brief reasoning (3–6 sentences). Be honest about uncertainty. Enumerate the assumptions you had to make.
- After your numeric answer, list the assumptions you had to make to produce it. The list is the point of this question.

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

The question: What was our total revenue in 2025?
```

## What to expect at this layer

The true answer is **$8,024.00** (sum of `Col014` across the 15 `COMPLETED` orders dated in 2025). The LLM may produce that number, but to do so it has to make at least three guesses:

1. **What column is a money amount?** `Col014` *looks* like a USD amount from value magnitudes, but no column is labeled. The LLM has to infer it from value patterns.
2. **What is "revenue"?** Not defined in the artifacts. The LLM has to invent a definition. Should `RETURNED` rows count? The data doesn't say. A naïve LLM that sums every row gets $8,064; one that excludes `RETURNED` lands at $8,024.
3. **What is "2025"?** Calendar year on `Col012` (date-shaped)? Fiscal year? The LLM has to assume.

A strong LLM under the closed-loop constraint will produce the number with at least three labeled caveats. A weak one will produce a confident answer that hides the assumptions. Either way: the answer is **guessing dressed as analysis**.

The right LLM behavior at this layer is *visibly hedging*. If the LLM ever says *"Col014 probably means a currency amount,"* or *"I'll assume RETURNED rows are excluded,"* — that's the demo working.

## The gap this layer reveals

This is the first move in the demo's arc. The lakehouse has the data and gets the right answer, but **only by guessing**:

- No column is named. Every interpretation is a value-pattern guess.
- No metric is defined. "Revenue" means whatever the LLM decides it means today.
- No filter intent is declared. `RETURNED` could mean "subtract this from revenue" or "ignore this row" — pick your reading.

That's three layers of guesswork on a question that should have one answer. The **semantic layer** is the cure: it gives every column a business name, gives every metric a formal definition, and gives every filter a documented intent. With it, the LLM stops guessing and starts citing.

Next demo: [02-semantic-prompt.md](02-semantic-prompt.md) — the same pattern, but the question is harder. Watch what the semantic layer fixes, and what it doesn't.

## Compare against

- Pre-baked transcript of this layer's response: [01-lakehouse-revenue.md](../transcripts/01-lakehouse-revenue.md).

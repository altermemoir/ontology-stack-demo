# Layer 1 transcript — "What was our total revenue in 2025?"

The lakehouse demo. The LLM has the five raw CSVs (inlined in [`../live-demo/01-lakehouse-prompt.md`](../live-demo/01-lakehouse-prompt.md)) and nothing else: opaque column headers (`Col001` - `Col031`), no documentation, no codelists, no schema. We ask it a question that should be trivial — last year's revenue — and watch how much guesswork it takes to get there.

**Ground truth:** **$8,024.00** — sum of `Col014` across the 15 rows of `orders_raw.csv` where `Col013 = "COMPLETED"` and `Col012` falls in calendar year 2025.

---

## What the LLM sees

The five CSVs verbatim. No documentation. The prompt explicitly tells it (per the closed-loop constraint) not to import outside knowledge.

## Simulated LLM response

> Looking at `orders_raw.csv` I see six unlabelled columns `Col010` - `Col015`. From the value patterns:
>
> - `Col010` looks like an order identifier (`O0001`, `O0002`, ...).
> - `Col011` looks like a customer identifier (values like `C001` that also appear in `customers_raw.csv` Col001).
> - `Col012` is date-formatted.
> - `Col013` contains the string values `COMPLETED` and `RETURNED`.
> - `Col014` is decimal-valued, magnitudes in the hundreds — *probably* a currency amount.
> - `Col015` is a two-letter code matching the country column on customers.
>
> To answer "total revenue in 2025" I have to commit to three guesses, none of which the data documents:
>
> 1. **`Col014` is the order's monetary amount.** I have no column name and no currency symbol; the magnitudes are consistent with USD given the SKU prices in `products_raw.csv` (e.g. SKU-1007 list price `349.00`, and `Col014` for orders containing it is in that neighborhood).
> 2. **`Col013 = "COMPLETED"` is the correct filter for "revenue."** I see one `RETURNED` row (O0004, Col014 = `40.00`). It is plausible that "revenue" should subtract returns, but I have no companion sign convention. I will exclude `RETURNED` rows from the sum rather than negate them, which is the conventional reading — but it is a reading, not a derivation.
> 3. **`Col012` is the order date and "2025" means the calendar year.** Both are guesses; neither is documented.
>
> Applying these guesses: filter `orders_raw` to rows where `Col012` starts with `2025-` and `Col013 = "COMPLETED"`, sum `Col014`. That gives 15 rows summing to **$8,024.00**.
>
> Caveats I want to flag explicitly:
>
> - I have inferred everything from column position and value patterns. The artifacts contain zero metadata.
> - A different reader could plausibly include the `RETURNED` row, or interpret `Col014` as a different currency, or read `Col012` as fiscal year. Each of those readings produces a different number.
> - There is no shared definition of "revenue" anywhere in the data. I am the only author of that definition in this answer.
>
> If you want this number to be the company's reported 2025 revenue, the next step is not "trust this answer" — it is "give me a metric definition and a schema document, and I can produce it without authoring the definition myself."

## What this response demonstrates

The LLM got the **right number**. It also produced **three labeled guesses** to get there. That's the demo's point. The right number isn't valuable if the path to it cannot be reproduced by another reader, another tool, or the same LLM tomorrow. Every business question that touches a column name or a status code is currently at the mercy of whichever interpretation the LLM picks first.

A weaker LLM at this layer fails differently — it produces $8,024 with a confident tone and no acknowledgement that any of the three steps were inferences. That outcome is *worse than declining*: it's a trustworthiness illusion. The right Layer-1 behavior, under the closed-loop constraint, is visibly hedging.

## The gap this layer reveals

| What's missing | What would fix it |
| --- | --- |
| No column names. | A semantic-layer column mapping (`Col014` → `order_amount_usd`). |
| No metric definition. | A named metric (`total_revenue`) with a published formula. |
| No filter intent. | A documented codelist for `Col013` (`COMPLETED` = "Fulfilled and paid", `RETURNED` = "revenue reversed"). |

All three live in a **semantic layer**. That's what the next demo adds.

## Next

The next transcript — [`02-semantic-copurchase.md`](02-semantic-copurchase.md) — adds the semantic layer and asks a question that the layer *cannot* answer cleanly. We watch the LLM compose a multi-CTE self-join from foreign-key declarations to answer a question that is structurally a graph.

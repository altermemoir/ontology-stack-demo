# live-demo — four demos, one arc

Four paste-and-run prompt packs. **One question per layer.** Each question is chosen so that the layer you're on *cannot answer it cleanly* — and the next layer up is exactly what would fix the gap. Run them in order and you watch the LLM's behavior shift, layer by layer, from guessing to citing.

## The arc

| # | Layer | Question | What this layer alone shows | Why the next layer is needed |
| --- | --- | --- | --- | --- |
| 1 | **Lakehouse** | "What was our total revenue in 2025?" | LLM has to guess what `Col014` means, what "revenue" means, and whether to include returns. | A **semantic layer** would name every column and define the metric. |
| 2 | **+ Semantic Layer** | "Which products are most frequently purchased together?" | LLM authors a multi-CTE self-join from the foreign keys. Works, but fragile, and the LLM is the query author. | A **knowledge graph** would replace the self-join with a two-edge traversal. |
| 3 | **+ Knowledge Graph** | "Which customers are at risk of churning?" | The data is connected — `lastOrderDate` and `openComplaintCount` sit right on each customer node — but no class for `ChurnRiskCustomer`. LLM invents the rule. | An **ontology** would define the class and the rule formally. |
| 4 | **+ Ontology** | "Which VIP customers cannot receive a promotional email under their region's marketing rules?" | LLM cites `nw:VIPCustomer`, `nw:RestrictedMarketingRegion`, and the `mayReceivePromoEmail` SHACL rule. Returns **C001 + C008** with a full audit trail. | (None — this is the payoff.) |

The diagonal is the demo's central point. Each layer alone hits a wall that the next layer is purpose-built to remove. By the time you reach the ontology, the LLM has stopped *inventing* and started *deriving*.

## How to run the demo (15–20 minutes)

1. Open four browser tabs of any frontier LLM (ChatGPT, Claude, Gemini — strong instruction-following matters more than provider).
2. **In tab 1** — fresh conversation. Open [`01-lakehouse-prompt.md`](01-lakehouse-prompt.md). Copy the code block (system prompt + 5 CSVs + the question) and paste it. Read the LLM's answer aloud, paying special attention to the *assumptions* it had to make.
3. **In tab 2** — fresh conversation. Repeat with [`02-semantic-prompt.md`](02-semantic-prompt.md). Watch the LLM compose a SQL self-join for a graph-shaped question. Note the cost of being the query author.
4. **In tab 3** — fresh conversation. Repeat with [`03-kg-prompt.md`](03-kg-prompt.md). Watch the LLM either invent a churn rule or (preferred under closed-loop constraint) decline and name the gap.
5. **In tab 4** — fresh conversation. Repeat with [`04-ontology-prompt.md`](04-ontology-prompt.md). Watch the LLM **cite by name** and return C001 + C008 with no hedging.

**Do not reuse a chat across layers.** Each demo's punch depends on the LLM not having seen the higher layers' definitions in its context. Fresh conversation per layer keeps the experiment clean.

The longest pack (Layer 4) is ~10–12K tokens; comfortably under any frontier-model limit.

## Cheat sheet — expected answers

| Layer | Question | Expected answer | What the LLM should also do |
| --- | --- | --- | --- |
| 1. Lakehouse | 2025 revenue | **$8,024.00** | Visibly hedge: label every assumption (which column is money, which status to exclude, what year means) as a guess. |
| 2. + Semantic | Most-co-purchased pairs | Three-way tie at 2 co-occurrences: **(SKU-1007 + SKU-1002)**, **(SKU-1006 + SKU-1002)**, **(SKU-1004 + SKU-1003)** | Show the self-join SQL. The shape of the query is the point. |
| 3. + KG | Customers at churn risk | **C003 Yuki Tanaka**, **C009 Klaus Weber** | Either explicitly label the rule as "my heuristic, not from the artifacts," OR decline and name what would unblock the question (i.e. a `ChurnRiskCustomer` class). |
| 4. + Ontology | VIPs forbidden from promo email | **C001 Anna Schmidt**, **C008 Priya Patel** | Cite `nw:VIPCustomer`, `nw:RestrictedMarketingRegion`, and `nw:MayReceivePromoEmailRule_Forbid` by name. Explain why C002 (opted in) and C004 (US not restricted) are exempt. |

## The qualitative shift to watch for

The number-of-correct-customers metric is not the point. The **voice** of the LLM is the point.

- **Layer 1** voice: *"I'll assume Col014 is currency..."* *"My heuristic is to exclude RETURNED..."* *"This is a guess from value patterns..."*
- **Layer 2** voice: *"Per the semantic model's metric definition..."* But also: *"There is no co-purchase metric, so I'll compose..."*
- **Layer 3** voice: *"The KG attaches `lastOrderDate` directly to each customer, so..."* But also: *"No class for `ChurnRiskCustomer` exists in the artifacts. My heuristic is..."* or (better) *"I will not invent the rule; the artifacts don't define it."*
- **Layer 4** voice: *"Per `nw:VIPCustomer` (lifetime ≥ $1500)..."* *"Applying `nw:MayReceivePromoEmailRule_Forbid`..."* *"By definition..."*

That shift — from *inventing* to *citing* — is the whole story.

## What variability to expect

Live LLM behavior varies between runs and between models. The demo's value is the **direction of progress** as you climb layers, not exact wording. Each pack includes a `CLOSED-LOOP CONSTRAINT` block that forbids the LLM from browsing the web or invoking training-corpus knowledge about regulations, geography, or external ontologies. Under that constraint, frontier LLMs typically behave like the cheat sheet above.

A demo *failure mode* to watch for: a Layer 1 LLM that confidently produces $8,024 without hedging — that means it's slipping past the closed-loop constraint and treating "revenue" as a known business term. Similarly, a Layer 3 LLM that confidently returns C003 + C009 with no acknowledgement that the rule is its own invention is leaking. Re-roll in a stricter environment if you see it.

## Why fresh conversations per layer?

Reusing a chat across layers leaks context: once the LLM has seen the semantic layer or the ontology, it cannot un-see them, and Layer 1's response will quietly use definitions it shouldn't have. A fresh conversation per layer keeps the experiment honest.

## When to use this vs. the pre-baked transcripts

| Use this folder | Use [`../transcripts/`](../transcripts/) |
| --- | --- |
| You have Wi-Fi and an LLM tab open. | You're presenting from a hotel-room projector. |
| You want live, slightly-non-deterministic responses. | You want a deterministic walkthrough every time. |
| Audience is engineers / AI-curious. | Audience is execs who want a clean narrative. |
| You want to highlight that "real LLMs really do this." | You want to highlight the *pattern* without rehearsal risk. |

Both paths reach the same destination. The transcripts are your safety net; the live demo is your truth.

## Maintenance note

The prompt packs are the source of truth for the demo data. Each pack inlines the artifacts for its layer and every layer below it (CSVs, YAML, Turtle, OWL/SHACL), so the same data appears in more than one pack. If you change an artifact:

1. Mirror the change in every pack that includes it (a CSV edit touches all four packs).
2. Re-check the expected answers in the cheat sheet above.
3. Update the matching transcript in [`../transcripts/`](../transcripts/).
4. Update any slide in the deck that quotes the data (slides 5, 8, 12, 14, and 18–21).

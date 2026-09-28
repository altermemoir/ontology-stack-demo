# Transcripts — four demos, one arc

Pre-baked LLM responses for the demo. **One question per layer.** Each question is chosen so the layer it's posed at *cannot answer it cleanly* — and the next layer up is exactly what would fix the gap. Read them in order; the arc is the message.

For the **live** version of this same walkthrough — paste-and-run prompt packs you can actually run against a frontier LLM — see [`../live-demo/`](../live-demo/). The transcripts in this folder are the deterministic fallback for when Wi-Fi, API quotas, or projector demos make a live run risky.

| File | Layer | Question | What the layer alone shows | The gap revealed |
| --- | --- | --- | --- | --- |
| [01-lakehouse-revenue.md](01-lakehouse-revenue.md) | Lakehouse | "What was our total revenue in 2025?" | LLM gets $8,024 by guessing what `Col014` means, what counts as "revenue," and which rows to exclude. | Needs a **semantic layer**. |
| [02-semantic-copurchase.md](02-semantic-copurchase.md) | + Semantic Layer | "Which products are most frequently purchased together?" | LLM authors a multi-CTE self-join from foreign-key declarations. Works, but fragile. | Needs a **knowledge graph**. |
| [03-kg-churn.md](03-kg-churn.md) | + Knowledge Graph | "Which customers are at risk of churning?" | Every fact the LLM needs is on each customer node, but no class for `ChurnRiskCustomer` exists. LLM either declines or invents the rule. | Needs an **ontology**. |
| [04-ontology-promo.md](04-ontology-promo.md) | + Ontology | "Which VIP customers cannot receive promotional email under their region's marketing rules?" | LLM cites `nw:VIPCustomer`, `nw:RestrictedMarketingRegion`, and `nw:MayReceivePromoEmailRule_Forbid`. Returns **C001 + C008** with audit trail. | None — this is the payoff. The ontology answers a compliance-sensitive question in a form the company can defend. |

The transcripts are hand-authored to be representative of what a frontier model produces when given each layer's context under a closed-loop constraint. They are not literal API outputs.

## Demo flow

1. **Open transcript 1** (lakehouse). Walk through the response. The point: even *simple* questions like "what was our revenue?" require the LLM to guess column meanings, definitions, and filter intent. Three guesses stacked. The lakehouse alone gives you guessing dressed as analysis.
2. **Open transcript 2** (+ semantic layer). The point: metrics solved Q1 cleanly. But the moment we ask a *relational* question (co-purchase), the LLM has to author SQL. The semantic layer is excellent for "metric over dimension" but inadequate for "graph over edges."
3. **Open transcript 3** (+ KG). The point: graph traversal made co-purchase natural, and made churn-risk *data* available on a single hop. But "at risk" is a concept the KG doesn't know — there's no class for it. The LLM is back to inventing the rule. Connection ≠ meaning.
4. **Open transcript 4** (+ ontology). The point: classes and SHACL rules turn the LLM from a definition-author into a definition-reader. Q3 — the compliance question — derives. Every step cites a named axiom or rule. The same query rederives identically tomorrow, next quarter, by anyone.

## What to say at the end

> "Lakehouses store data. Semantic layers name it. Knowledge graphs connect it. Ontologies *mean* something. Each layer has its own jurisdiction. We saw one question per layer where the layer's strengths and limits are both visible — revenue is solved by metrics, co-purchase is solved by graph edges, churn-risk and promo-policy are solved by formal classes and rules. The value compounds because each layer earns its keep on a different *shape* of question."

## The qualitative shift to watch for

The number of correct customers, dollars, or SKUs is not the point. The **voice** of the LLM is the point.

- **Lakehouse voice:** *"I'll assume Col014 is currency..."* *"My heuristic, not from the artifacts..."* *"A different reader could pick differently."*
- **Semantic voice:** *"Per the metric definition..."* But also: *"There is no co-purchase metric, so I'll compose..."*
- **KG voice:** *"The graph attaches lastOrderDate directly to each customer."* But also: *"No class for ChurnRiskCustomer exists in the artifacts. I will not invent the rule."*
- **Ontology voice:** *"Per `nw:VIPCustomer` (lifetime ≥ $1500)..."* *"Applying `nw:MayReceivePromoEmailRule_Forbid`..."* *"By definition..."*

That progression — from *inventing* to *citing* — is the whole story.
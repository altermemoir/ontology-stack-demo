# Layer 3 transcript — "Which customers are at risk of churning?"

The knowledge-graph demo. The LLM has the CSVs, the semantic model, *and* the full RDF knowledge graph from [`../03-knowledge-graph/`](../03-knowledge-graph/). Each `nw:Customer` carries pre-computed aggregate facts: `nw:totalLifetimeSpend`, `nw:lastOrderDate`, `nw:openComplaintCount`. We ask "who's at risk of churning?" and watch the LLM hit the wall the ontology is built for.

**Ground truth (per the ontology, which the LLM does *not* yet see):** **C003 Yuki Tanaka, C009 Klaus Weber**. Both have `lastOrderDate` more than 180 days before today (2026-06-01) AND `openComplaintCount >= 1`. Both conditions are required — C007 Carlos has no recent order but his complaint is *resolved*, so he is excluded.

---

## What the LLM sees

CSVs + semantic model + KG. Every customer's facts are on a single node. There is no `nw:ChurnRiskCustomer` class anywhere in the artifacts. There is no SHACL rule about churn. The word "churn" appears only in the question.

## Simulated LLM response (closed-loop-compliant path — preferred)

> Each `nw:Customer` in the KG carries `nw:lastOrderDate` and `nw:openComplaintCount` directly:
>
> | customer | lastOrderDate | openComplaintCount |
> |---|---|---|
> | C001 Anna | 2025-08-30 | 0 |
> | C002 Marco | 2025-11-20 | 0 |
> | C003 Yuki | 2024-11-05 | 1 |
> | C004 Sarah | 2026-04-12 | 0 |
> | C005 Hans | 2025-09-15 | 0 |
> | C006 Emma | 2026-03-30 | 1 |
> | C007 Carlos | 2025-09-18 | 0 |
> | C008 Priya | 2025-11-25 | 0 |
> | C009 Klaus | 2024-08-20 | 1 |
> | C010 Sophie | 2026-02-14 | 0 |
>
> The data needed to characterize a "churn risk" customer is right here — I do not need to aggregate, join, or compose. The graph has done that work.
>
> But: I cannot find a class `nw:ChurnRiskCustomer` in the artifacts. I cannot find a SHACL rule or any axiom that defines *when* the conditions `lastOrderDate < threshold` and `openComplaintCount >= 1` should trigger membership in some "at risk" set. The KG is a connected pile of facts; it does not tell me which combination of facts *means* "at risk."
>
> Under the closed-loop constraint, I am not going to invent a rule. The right answer is to identify what's missing: this question needs a `nw:ChurnRiskCustomer` class (or equivalent) defined at a layer that owns class hierarchy and rules. The KG is not that layer.
>
> If you give me a definition — e.g. "180-day cutoff and at least one open complaint" — I can apply it. Without one, the closest I can offer is: "candidates worth examining if you supply a definition" — customers with a stale lastOrderDate and/or an open complaint are C003, C006, and C009. But classification belongs to the next layer.

## Simulated LLM response (rule-inventing path — also seen in practice)

> The KG attaches `lastOrderDate` and `openComplaintCount` directly to each customer, so no aggregation is needed. The question, however, requires a rule for "at risk of churning" that I do not find in the artifacts. **My heuristic, not from the artifacts:** combine the two signals — flag customers whose last order is more than 180 days old AND who have at least one open complaint. Today is 2026-06-01, so the date cutoff is 2025-12-03.
>
> - **C003 Yuki Tanaka** — lastOrderDate 2024-11-05 (before cutoff ✓), openComplaintCount 1 ✓ → **flag**
> - **C009 Klaus Weber** — lastOrderDate 2024-08-20 (before cutoff ✓), openComplaintCount 1 ✓ → **flag**
> - **C007 Carlos Garcia** — lastOrderDate 2025-09-18 (after cutoff), openComplaintCount 0 → not flagged.
> - **C006 Emma Davies** — lastOrderDate 2026-03-30 (recent), openComplaintCount 1 → would be a candidate under a "recency-OR-complaint" rule but not under my AND rule.
>
> Result under *my invented rule*: **C003, C009.** A different rule (e.g. "no order in 180 days OR an open complaint") would also flag C006 and possibly more. The answer is hostage to the rule I picked.

## What this response demonstrates

The KG fixed everything it can fix:

- The **data** for churn is available on a single hop. No SQL aggregation. No `GROUP BY customer_id`. The graph pre-computed it.
- The **traversal** is trivial. Each customer node carries the facts.
- The **connection** is dense. Region, complaints, order history all hang off the customer node.

What the KG **cannot** do is tell the LLM what those facts *mean*. "At risk of churning" is a concept — and concepts live in the ontology. The two response paths illustrate the two ways an LLM can fail:

- **Compliant path:** decline and name the gap. Honest. Pushes the work to the layer that should own it.
- **Inventing path:** pick a threshold, get a defensible-looking list, ship it. The list happens to match company policy *only because the LLM picked the right threshold and signal mix on this run*. A different LLM, a different prompt, a different day will pick differently. There is no audit trail.

Both are useful demo outcomes. The compliant path makes the missing-layer case explicit. The inventing path shows what "guessing dressed as analysis" looks like at a layer that *should* know better — and that's what the ontology fixes.

## The gap this layer reveals

| What's still missing | What would fix it |
| --- | --- |
| No class for `ChurnRiskCustomer`. | An OWL class definition with a `rdfs:comment` describing the policy. |
| No rule for triggering membership. | A SHACL rule combining the two conditions. |
| No authority for what counts as "at risk." | The ontology, with the rule under version control. |

The pattern: the KG owns *connection*. The ontology owns *meaning*. Two complementary layers, two different jobs.

A note on `legacyTier`: every customer node carries `nw:legacyTier` (P/G/S/B) — the lakehouse `Col007` mapped through. If the LLM is tempted to use that to identify "high-value" customers at risk, the KG does not stop it from doing so. The KG faithfully carries the legacy field; it does not carry the *policy* that says "legacyTier is not authoritative." That policy is what the ontology adds.

## Next

The next transcript — [`04-ontology-promo.md`](04-ontology-promo.md) — adds the ontology and asks a harder, business-loaded question: "which VIP customers cannot receive promo email under their region's marketing rules?" This is the payoff of the stack. Watch the LLM stop hedging and start citing.

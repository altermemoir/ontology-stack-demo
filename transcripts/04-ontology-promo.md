# Layer 4 transcript — "Which VIP customers cannot receive promo email under their region's marketing rules?"

The ontology demo. The payoff. The LLM has the full stack: CSVs, semantic model, knowledge graph, **and** the OWL classes + SHACL rules from [`../04-ontology/`](../04-ontology/). We ask a question that combines two derived class memberships and a policy rule, and watch the LLM stop *inventing* and start *citing*.

**Ground truth:** **C001 Anna Schmidt** and **C008 Priya Patel**. VIPs (lifetime spend ≥ $1500) who live in restricted-marketing regions (DE and GB respectively) and have not opted in to promotional email.

---

## What the LLM sees

Everything the prior demos had, plus:

- `retail.ttl` — defines `nw:VIPCustomer`, `nw:ChurnRiskCustomer`, `nw:RestrictedMarketingRegion` as OWL classes; asserts which Regions are `RestrictedMarketingRegion` (DE, FR, IT, ES, GB); declares the `nw:mayReceivePromoEmail` derived property.
- `retail-shapes.ttl` — SHACL rules that turn the class definitions into machine-derivable inferences: `nw:VIPCustomerRule` (lifetime ≥ 1500 ⇒ `a nw:VIPCustomer`), `nw:MayReceivePromoEmailRule_Forbid` (livesIn `nw:RestrictedMarketingRegion` AND `promoOptIn = false` ⇒ `mayReceivePromoEmail false`), and a complementary `_Allow` rule.

## Simulated LLM response

> The question composes two ontology classes and a SHACL-derived property. I will trace it step by step, citing each artifact.
>
> **Step 1 — apply `nw:VIPCustomerRule` to identify VIPs.**
>
> The rule (from `retail-shapes.ttl`) classifies any `nw:Customer` whose `nw:totalLifetimeSpend` is ≥ 1500 as `nw:VIPCustomer`. Walking the customer nodes in the KG:
>
> | customer | totalLifetimeSpend | VIPCustomer? |
> |---|---:|---|
> | C001 Anna Schmidt | $1,575.00 | **yes** |
> | C002 Marco Rossi | $1,605.00 | **yes** |
> | C003 Yuki Tanaka | $698.00 | no |
> | C004 Sarah Johnson | $1,949.00 | **yes** |
> | C005 Hans Mueller | $790.00 | no |
> | C006 Emma Davies | $225.00 | no |
> | C007 Carlos Garcia | $25.00 | no |
> | C008 Priya Patel | $2,119.00 | **yes** |
> | C009 Klaus Weber | $349.00 | no |
> | C010 Sophie Laurent | $220.00 | no |
>
> VIPs are **C001, C002, C004, C008**. I note that C003 and C009 carry `nw:legacyTier "P"` (Platinum in the legacy CRM), but per the `rdfs:comment` on `nw:VIPCustomer` — *"Authoritative — supersedes legacyTier"* — the legacy field is not the basis for VIP. The ontology is explicit on this point.
>
> **Step 2 — identify `nw:RestrictedMarketingRegion` members.**
>
> Asserted directly in `retail.ttl`:
>
> - DE, FR, IT, ES, GB → `RestrictedMarketingRegion`
> - US, JP → not `RestrictedMarketingRegion`
>
> No inference required; these are direct class memberships in the ontology.
>
> **Step 3 — apply `nw:MayReceivePromoEmailRule_Forbid` to each VIP.**
>
> The rule fires `mayReceivePromoEmail = false` iff `livesIn ∈ RestrictedMarketingRegion` AND `promoOptIn = false`.
>
> | VIP | livesIn | restricted? | promoOptIn | mayReceivePromoEmail |
> |---|---|---|---|---|
> | C001 Anna | DE | **yes** | false | **false** ← Forbid rule fires |
> | C002 Marco | IT | yes | **true** | true ← Allow rule fires (opted in) |
> | C004 Sarah | US | **no** | false | true ← Allow rule fires (not in restricted region) |
> | C008 Priya | GB | **yes** | false | **false** ← Forbid rule fires |
>
> **Result: VIPs forbidden from promo email are C001 Anna Schmidt and C008 Priya Patel.**
>
> Notes on the two exemptions:
>
> - **C002 (Marco, Italy)** is a VIP in a `RestrictedMarketingRegion`, but his `promoOptIn` is true. The Forbid rule requires both conditions; opting in releases him.
> - **C004 (Sarah, United States)** is a VIP with `promoOptIn = false`, but US is not a `RestrictedMarketingRegion`. The Forbid rule's first condition fails.
>
> Every step in this answer cites a specific axiom or rule by IRI. A SHACL validator running over the KG produces the same `nw:mayReceivePromoEmail false` triples for C001 and C008 with no LLM in the loop. The LLM is reading derived facts here, not inventing rules.

## What this response demonstrates

The qualitative shift the whole demo is built around. The LLM stops saying *"I'll assume,"* *"my heuristic is,"* *"a different reader could pick differently"* — and starts saying *"per `nw:VIPCustomer`,"* *"applying `nw:MayReceivePromoEmailRule_Forbid`,"* *"by definition."*

Three properties of this response that the lower-layer responses lacked:

1. **Authoritative.** The class definitions and rules are in version-controlled files. The next LLM, the next analyst, the next compliance auditor reaches the same answer from the same artifacts.
2. **Auditable.** Every customer in the answer has a trace: which class they belong to (cite the class), why they belong to it (cite the rule's condition), which property fired (cite the SHACL rule's predicate). When the compliance team asks *"why was Anna not emailed?"*, the answer is one IRI lookup, not a conversation with whoever wrote the marketing query.
3. **Reproducible without an LLM.** A SHACL validator does not need a frontier model. Run the rules on the KG and the same triples appear. The LLM is now optional infrastructure — useful for natural-language access but not load-bearing for correctness.

## The business problem this layer solves

This is the question that has compliance dollars attached. Sending unsolicited promotional email to a non-opted-in resident of a restricted region is a fine, per recipient, under regional marketing rules. At scale, the math becomes real.

Three places this rule could have lived, three different operational outcomes:

| Where the rule lives | What happens when... |
|---|---|
| **In a marketing manager's head** | ...they leave the company. The rule silently regresses. The next manager picks a different threshold or forgets a region. |
| **Hardcoded into the email-sending service** | ...the rule changes. You update the service. You also update the BI dashboard, the audit query, the segmentation tool, and the LLM analytics assistant — and each one has its own bugs. The rule's definition is duplicated across systems. |
| **In the ontology** | ...the rule changes. You update one file. Every downstream consumer — email service, audit query, LLM, dashboard — re-derives from the new authoritative definition. Change once, propagate everywhere. |

That last property is what makes the ontology a *capability*. The answer to "which VIPs cannot receive promo email?" is not what's valuable; you could get it with enough careful SQL. What's valuable is that the answer **rederives identically** every time anyone — human or machine — asks the question against the live data. The LLM stopped authoring policy and started reading it.

## Why the lower layers cannot do this

For completeness, the failure mode at each lower layer:

- **Lakehouse alone:** has to guess that `Col007 = "P"` means VIP (incorrect — legacy field, has drifted). Has to guess from general knowledge which regions are restricted (closed-loop constraint forbids this). Has to invent the opt-in policy. Three layers of guesses stacked; a strong LLM should decline outright.
- **+ Semantic layer:** knows `customer_tier` is legacy and *not* authoritative (the YAML warns it), but doesn't know what *is* authoritative for VIP. Doesn't have a region-restriction list. Doesn't have a policy rule. Under closed-loop constraint, declines.
- **+ Knowledge graph:** has every data point (lifetime spend, region, opt-in flag) on each customer node, but no class for VIP, no marker for restricted regions, no policy rule. Under closed-loop constraint, declines.
- **+ Ontology:** the three missing pieces — class, region markers, SHACL rule — are now on the page. The answer derives.

The whole stack is in service of this last bullet.

## End of the arc

This is the last transcript. The four together tell one story:

1. The **lakehouse** had the data but no names, no definitions. We answered $8,024 by guessing.
2. The **semantic layer** added names and metrics — and revealed that pure metrics can't express relational shapes. Co-purchase forced us to author SQL.
3. The **knowledge graph** added typed edges and pre-computed aggregates — and revealed that connection doesn't supply meaning. Churn risk needed a definition the artifacts didn't carry.
4. The **ontology** added classes, axioms, and SHACL rules — and answered the question that the company would actually defend in an audit.

The diagonal across the demos is the entire point: each layer hits a wall the next layer is built to remove. None of the layers is a vanity layer. None replaces the one below. The value compounds.

> *Lakehouses store data. Semantic layers name it. Knowledge graphs connect it. Ontologies* mean *something. Each layer has its own jurisdiction, and you need all four to put a compliance-sensitive question through a defensible pipeline.*

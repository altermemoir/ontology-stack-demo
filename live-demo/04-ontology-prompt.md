# Layer 4 — Lakehouse + Semantic Layer + Knowledge Graph + Ontology (live demo prompt)

**What this demo shows:** the question we ask here — *"which VIP customers cannot receive a promotional email under their region's marketing rules?"* — has compliance risk attached. Sending the wrong promo to the wrong customer in the wrong region is a regulatory bug with real dollar consequences. The lower layers cannot answer this question without inventing definitions. The ontology can — and it answers in a form the company can defend in an audit. **This is what the whole stack was for.**

Paste these blocks into a **fresh** LLM conversation. Do not reuse the chat from Layers 1, 2, or 3.

## Step 1 — paste this as your only message

```text
You are a senior data analyst at NorthWind Outdoor, a fictional online outdoor-gear retailer. Today's date is 2026-06-01.

You have been given the company's full data and knowledge stack: (a) raw lakehouse CSVs (with opaque headers Col001 - Col031 — meaning recoverable only via the semantic model), (b) a semantic-model YAML, (c) a knowledge graph in Turtle/RDF, (d) an OWL ontology defining business classes (Customer, VIPCustomer, ChurnRiskCustomer, RestrictedMarketingRegion) with axioms, and (e) SHACL rules that derive class memberships and policy properties (mayReceivePromoEmail). Each artifact is delimited by a "=== File: <name> ===" marker.

CLOSED-LOOP CONSTRAINT (read this first):
This is a closed-loop evaluation. You may use ONLY the artifacts pasted in this message and your reasoning over them. Nothing else. Specifically, you must NOT:
- browse the web, search, or call any retrieval tool;
- invoke training-corpus knowledge about regulations (GDPR, UK GDPR, CCPA, ePrivacy, CAN-SPAM, etc.), geography (which countries are in the EU/EEA, etc.), industry conventions ("typical churn rules," "typical VIP thresholds," "what most retailers do"), or external ontologies (FIBO, schema.org, GoodRelations, etc.) you may have seen elsewhere;
- silently use general knowledge to fill a gap. If you find yourself reaching for it, stop and say "this requires X, which the artifacts above do not provide."

If a question requires information not in the artifacts, the correct response is to identify the gap explicitly and stop — not to guess from outside knowledge.

Layer-specific guidance:
- The OWL classes and SHACL rules in the artifacts above are AUTHORITATIVE for any concept they define (VIP, ChurnRisk, RestrictedMarketingRegion, mayReceivePromoEmail). Apply this ontology's definitions and rules — do not invent your own and do not reference any other ontology.
- Cite the relevant class or rule by name in your answer (e.g. "per nw:VIPCustomer," "applying nw:MayReceivePromoEmailRule_Forbid").
- Show the intermediate steps: who is a VIPCustomer, who livesIn a RestrictedMarketingRegion, who has promoOptIn=false. The trace matters as much as the final list.

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

=== File: semantic_model.yml (excerpt) ===
metrics:
  - name: total_revenue
    formula: SUM(order.order_amount_usd) WHERE order.order_status = 'COMPLETED'

note: customer_tier (legacy CRM) is NOT authoritative for VIP — the ontology owns that.

=== File: instances.ttl (knowledge graph, customer + region facts) ===
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix nw:   <http://example.org/northwind/> .

# Regions
nw:Region_DE a nw:Region ; nw:countryCode "DE" .
nw:Region_FR a nw:Region ; nw:countryCode "FR" .
nw:Region_IT a nw:Region ; nw:countryCode "IT" .
nw:Region_ES a nw:Region ; nw:countryCode "ES" .
nw:Region_GB a nw:Region ; nw:countryCode "GB" .
nw:Region_US a nw:Region ; nw:countryCode "US" .
nw:Region_JP a nw:Region ; nw:countryCode "JP" .

# Customers (with pre-computed aggregates from the orders/tickets data)
nw:C001 a nw:Customer ;
  nw:firstName "Anna" ; nw:lastName "Schmidt" ;
  nw:livesIn nw:Region_DE ; nw:legacyTier "P" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 1575.00 ; nw:lastOrderDate "2025-08-30"^^xsd:date ; nw:openComplaintCount 0 .

nw:C002 a nw:Customer ;
  nw:firstName "Marco" ; nw:lastName "Rossi" ;
  nw:livesIn nw:Region_IT ; nw:legacyTier "G" ; nw:promoOptIn true ;
  nw:totalLifetimeSpend 1605.00 ; nw:lastOrderDate "2025-11-20"^^xsd:date ; nw:openComplaintCount 0 .

nw:C003 a nw:Customer ;
  nw:firstName "Yuki" ; nw:lastName "Tanaka" ;
  nw:livesIn nw:Region_JP ; nw:legacyTier "P" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 698.00 ; nw:lastOrderDate "2024-11-05"^^xsd:date ; nw:openComplaintCount 1 .

nw:C004 a nw:Customer ;
  nw:firstName "Sarah" ; nw:lastName "Johnson" ;
  nw:livesIn nw:Region_US ; nw:legacyTier "G" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 1949.00 ; nw:lastOrderDate "2026-04-12"^^xsd:date ; nw:openComplaintCount 0 .

nw:C005 a nw:Customer ;
  nw:firstName "Hans" ; nw:lastName "Mueller" ;
  nw:livesIn nw:Region_DE ; nw:legacyTier "P" ; nw:promoOptIn true ;
  nw:totalLifetimeSpend 790.00 ; nw:lastOrderDate "2025-09-15"^^xsd:date ; nw:openComplaintCount 0 .

nw:C006 a nw:Customer ;
  nw:firstName "Emma" ; nw:lastName "Davies" ;
  nw:livesIn nw:Region_GB ; nw:legacyTier "S" ; nw:promoOptIn true ;
  nw:totalLifetimeSpend 225.00 ; nw:lastOrderDate "2026-03-30"^^xsd:date ; nw:openComplaintCount 1 .

nw:C007 a nw:Customer ;
  nw:firstName "Carlos" ; nw:lastName "Garcia" ;
  nw:livesIn nw:Region_ES ; nw:legacyTier "B" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 25.00 ; nw:lastOrderDate "2025-09-18"^^xsd:date ; nw:openComplaintCount 0 .

nw:C008 a nw:Customer ;
  nw:firstName "Priya" ; nw:lastName "Patel" ;
  nw:livesIn nw:Region_GB ; nw:legacyTier "G" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 2119.00 ; nw:lastOrderDate "2025-11-25"^^xsd:date ; nw:openComplaintCount 0 .

nw:C009 a nw:Customer ;
  nw:firstName "Klaus" ; nw:lastName "Weber" ;
  nw:livesIn nw:Region_DE ; nw:legacyTier "P" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 349.00 ; nw:lastOrderDate "2024-08-20"^^xsd:date ; nw:openComplaintCount 1 .

nw:C010 a nw:Customer ;
  nw:firstName "Sophie" ; nw:lastName "Laurent" ;
  nw:livesIn nw:Region_FR ; nw:legacyTier "S" ; nw:promoOptIn true ;
  nw:totalLifetimeSpend 220.00 ; nw:lastOrderDate "2026-02-14"^^xsd:date ; nw:openComplaintCount 0 .

=== File: retail.ttl (ontology — classes and axioms) ===
@prefix rdfs:  <http://www.w3.org/2000/01/rdf-schema#> .
@prefix owl:   <http://www.w3.org/2002/07/owl#> .
@prefix xsd:   <http://www.w3.org/2001/XMLSchema#> .
@prefix nw:    <http://example.org/northwind/> .

nw:Customer a owl:Class ; rdfs:label "Customer" .
nw:Order    a owl:Class ; rdfs:label "Order" .
nw:Region   a owl:Class ; rdfs:label "Region" .

nw:VIPCustomer a owl:Class ;
  rdfs:subClassOf nw:Customer ;
  rdfs:label "VIP Customer" ;
  rdfs:comment "A Customer whose totalLifetimeSpend is at least USD 1500. Authoritative — supersedes legacyTier." .

nw:ChurnRiskCustomer a owl:Class ;
  rdfs:subClassOf nw:Customer ;
  rdfs:label "Churn-Risk Customer" ;
  rdfs:comment "A Customer whose lastOrderDate is more than 180 days before today AND whose openComplaintCount is at least 1. Both conditions required." .

nw:RestrictedMarketingRegion a owl:Class ;
  rdfs:subClassOf nw:Region ;
  rdfs:comment "A Region in which promotional email may only be sent to customers with explicit prior opt-in." .

# Asserted membership in RestrictedMarketingRegion:
nw:Region_DE a nw:RestrictedMarketingRegion .
nw:Region_FR a nw:RestrictedMarketingRegion .
nw:Region_IT a nw:RestrictedMarketingRegion .
nw:Region_ES a nw:RestrictedMarketingRegion .
nw:Region_GB a nw:RestrictedMarketingRegion .
# Region_US and Region_JP are NOT RestrictedMarketingRegion.

nw:totalLifetimeSpend a owl:DatatypeProperty ; rdfs:domain nw:Customer ; rdfs:range xsd:decimal .
nw:lastOrderDate      a owl:DatatypeProperty ; rdfs:domain nw:Customer ; rdfs:range xsd:date .
nw:openComplaintCount a owl:DatatypeProperty ; rdfs:domain nw:Customer ; rdfs:range xsd:integer .
nw:promoOptIn         a owl:DatatypeProperty ; rdfs:domain nw:Customer ; rdfs:range xsd:boolean .
nw:livesIn            a owl:ObjectProperty   ; rdfs:domain nw:Customer ; rdfs:range nw:Region .

nw:mayReceivePromoEmail a owl:DatatypeProperty ;
  rdfs:domain nw:Customer ; rdfs:range xsd:boolean ;
  rdfs:comment "Derived. False iff customer livesIn a RestrictedMarketingRegion AND promoOptIn = false. Otherwise true. See SHACL rules." .

=== File: retail-shapes.ttl (SHACL rules) ===
@prefix sh:   <http://www.w3.org/ns/shacl#> .
@prefix rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix nw:   <http://example.org/northwind/> .

# Rule 1: Any Customer with totalLifetimeSpend >= 1500 is a VIPCustomer.
nw:VIPCustomerRule a sh:NodeShape ; sh:targetClass nw:Customer ;
  sh:rule [ a sh:TripleRule ;
    sh:subject sh:this ; sh:predicate rdf:type ; sh:object nw:VIPCustomer ;
    sh:condition [ sh:property [ sh:path nw:totalLifetimeSpend ; sh:minInclusive 1500 ] ]
  ] .

# Rule 2: mayReceivePromoEmail = false iff livesIn a RestrictedMarketingRegion AND promoOptIn = false.
nw:MayReceivePromoEmailRule_Forbid a sh:NodeShape ; sh:targetClass nw:Customer ;
  sh:rule [ a sh:TripleRule ;
    sh:subject sh:this ; sh:predicate nw:mayReceivePromoEmail ; sh:object false ;
    sh:condition [ sh:and (
      [ sh:property [ sh:path nw:livesIn ;    sh:class nw:RestrictedMarketingRegion ] ]
      [ sh:property [ sh:path nw:promoOptIn ; sh:hasValue false ] ]
    ) ]
  ] .

# Rule 3: mayReceivePromoEmail = true iff NOT livesIn a RestrictedMarketingRegion OR promoOptIn = true.
nw:MayReceivePromoEmailRule_Allow a sh:NodeShape ; sh:targetClass nw:Customer ;
  sh:rule [ a sh:TripleRule ;
    sh:subject sh:this ; sh:predicate nw:mayReceivePromoEmail ; sh:object true ;
    sh:condition [ sh:or (
      [ sh:property [ sh:path nw:livesIn ;    sh:not [ sh:class nw:RestrictedMarketingRegion ] ] ]
      [ sh:property [ sh:path nw:promoOptIn ; sh:hasValue true ] ]
    ) ]
  ] .

The question: Which VIP customers cannot receive a promotional email under their region's marketing rules?
```

## What to expect at this layer

The canonical answer: **C001 Anna Schmidt** and **C008 Priya Patel**.

Watch the LLM's behavior change from the prior three layers. There is no hedging. There is no "I'll assume." Instead the LLM should *cite*:

1. **Apply `nw:VIPCustomerRule` (totalLifetimeSpend ≥ 1500):**
   - C001 — $1,575 ✓
   - C002 — $1,605 ✓
   - C004 — $1,949 ✓
   - C008 — $2,119 ✓
   - All others below threshold.
   - **Note explicitly:** C003 (Yuki) and C009 (Klaus) are *coded* `legacyTier "P"` (Platinum) — but the ontology says `legacyTier` is *not* authoritative for VIP. The legacy field has drifted. The ontology is the arbiter.

2. **Identify `nw:RestrictedMarketingRegion` members:** DE, FR, IT, ES, GB. (US and JP are not.)

3. **Apply `nw:MayReceivePromoEmailRule_Forbid` to each VIP:**
   - C001 (livesIn DE — restricted; promoOptIn = false) → `mayReceivePromoEmail = false` ✗
   - C002 (livesIn IT — restricted; promoOptIn = **true**) → may receive (opted in) ✓
   - C004 (livesIn US — **not** restricted) → may receive (US is not a restricted region) ✓
   - C008 (livesIn GB — restricted; promoOptIn = false) → `mayReceivePromoEmail = false` ✗

4. **Result: C001 (Anna Schmidt) and C008 (Priya Patel).**

A strong LLM at this layer will name the classes (`nw:VIPCustomer`, `nw:RestrictedMarketingRegion`) and the rule (`nw:MayReceivePromoEmailRule_Forbid`) by their IRIs, not just by paraphrase. It will *also* explicitly call out the exemptions:
- *"C002 is exempt because he opted in."*
- *"C004 is exempt because US is not a `RestrictedMarketingRegion`."*

That explicit-exemption behavior is the qualitative tell. Watch for the phrase *"per the ontology,"* *"applying the SHACL rule,"* *"by definition,"* and watch the *absence* of *"I'll assume,"* *"in many companies,"* *"typically."*

## The business problem this layer solves

Sending the wrong promo to the wrong person in the wrong region is a regulatory bug. Under EU/UK marketing rules, sending unsolicited promotional email to a non-opted-in resident is a fine. The fines are per-recipient. At enough scale, that's a real number.

Three places this question could have been answered, three different outcomes:

| Where the rule lives | What happens when someone leaves |
|---|---|
| **In a marketing manager's head.** | The rule changes when they get a new job. Compliance regresses silently. |
| **In code, in the email-sending service.** | The rule is duplicated in every system that touches customers. Each duplicate has its own bugs. |
| **In the ontology.** | One authoritative file. One definition. Every downstream consumer — the LLM, the email service, the audit query, the compliance dashboard — derives from the same rule. Change the rule in one place; every consumer updates. |

The third option is what makes the ontology a *capability*, not paperwork. It's not that you couldn't get C001 and C008 by other means — you could, with enough careful SQL and enough good intentions. It's that the ontology is the only place where the answer is **derivable and defensible**:

- **Derivable:** a SHACL validator running on the KG produces the same `mayReceivePromoEmail = false` triples without an LLM in the loop. The LLM is reading derived facts, not authoring rules.
- **Defensible:** the rule is in a file, under version control, with a `rdfs:comment` describing its business intent. When the compliance team asks "why did you not email Anna?", the answer is one IRI lookup away.

## What this demo is *not*

It is not an argument that every business question needs an ontology. The previous three demos make that point: revenue is a metric layer's job, co-purchase is a graph layer's job, churn risk is *also* an ontology question but a simpler one. The ontology earns its keep on questions where **definitions matter and policies have consequences** — and this is one of them.

## End of the arc

This is the last demo. The four together tell one story:

1. The **lakehouse** has the data. It cannot answer cleanly without guessing what columns mean.
2. The **semantic layer** names the columns and defines the metrics. It cannot answer questions whose shape is *relational*.
3. The **knowledge graph** types the entities and connects them with edges. It cannot answer questions whose shape is *rule-based*.
4. The **ontology** defines the rules. It is the only layer where business meaning becomes machine-derivable and audit-defensible.

Each layer is additive. Each layer earns its keep on a different *shape* of question. None is a vanity layer. None replaces the one below.

## Compare against

- Pre-baked transcript of this layer's response: [04-ontology-promo.md](../transcripts/04-ontology-promo.md).

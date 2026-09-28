# Layer 3 — Lakehouse + Semantic Layer + Knowledge Graph (live demo prompt)

**What this demo shows:** the knowledge graph fixed the co-purchase question by replacing a self-join with a two-edge traversal. Same data, structurally simpler answer. But now we ask a different *kind* of question — "which customers are at risk of churning?" — and the KG hits its own wall. All the facts the LLM needs are sitting on each customer's node (last order date, open complaint count). But there's no class for `ChurnRiskCustomer`. There's no rule for *when* the conditions trigger membership. The KG knows the *facts*; it doesn't know the *concept*. So the LLM is back to inventing the definition. **This is the case for an ontology.**

Paste these blocks into a **fresh** LLM conversation. Do not reuse the chat from Layer 1 or 2.

## Step 1 — paste this as your only message

```text
You are a senior data analyst at NorthWind Outdoor, a fictional online outdoor-gear retailer. Today's date is 2026-06-01.

You have been given three artifacts: (a) the raw lakehouse CSVs (with opaque headers Col001 - Col031 — meaning is recoverable only via the semantic model), (b) a semantic-model YAML mapping each opaque column to a business name and defining metrics, and (c) a knowledge graph in Turtle/RDF format that instantiates customers, orders, products, regions, brands, and support tickets — and the relationships among them. Each artifact is delimited by a "=== File: <name> ===" marker.

CLOSED-LOOP CONSTRAINT (read this first):
This is a closed-loop evaluation. You may use ONLY the artifacts pasted in this message and your reasoning over them. Nothing else. Specifically, you must NOT:
- browse the web, search, or call any retrieval tool;
- invoke training-corpus knowledge about regulations, geography, industry conventions ("typical churn rules," "typical VIP thresholds," "what most retailers do"), or external ontologies you may have seen elsewhere;
- silently use general knowledge to fill a gap. If you find yourself reaching for it, stop and say "this requires X, which the artifacts above do not provide."

If a question requires information not in the artifacts, the correct response is to identify the gap explicitly and stop — not to guess from outside knowledge.

Layer-specific guidance:
- The semantic model defines metrics; treat them as authoritative.
- The knowledge graph contains pre-computed aggregate properties on each Customer (totalLifetimeSpend, lastOrderDate, openComplaintCount). These are facts. Use them; do not re-derive them.
- The KG provides typed entities and relationships (livesIn, placedBy, hasBrand, etc.) that make traversal easy.
- For concepts the artifacts do NOT formally define — i.e. there is no class definition, no SHACL rule, no axiom for the concept anywhere in the artifacts — say so explicitly. If you choose to offer a heuristic guess, label it "my heuristic, not from the artifacts." Prefer to decline and name what would unblock the question rather than invent a rule and pass it off as derived.
- Show brief reasoning.

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

=== File: semantic_model.yml (excerpt) ===
metrics:
  - name: total_revenue
    formula: SUM(order.order_amount_usd) WHERE order.order_status = 'COMPLETED'
  - name: open_ticket_count
    formula: COUNT(support_ticket.ticket_id) WHERE support_ticket.ticket_status = 'OPEN'
  - name: days_since_last_order
    formula: DATEDIFF(today, MAX(order.order_date)) WHERE order.order_status = 'COMPLETED' GROUP BY order.customer_id

note: customer_tier (legacy CRM field) is NOT authoritative for VIP status.
note: ChurnRiskCustomer is not a class defined at the semantic layer.

=== File: instances.ttl (knowledge graph) ===
@prefix rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix nw:   <http://example.org/northwind/> .

# Regions
nw:Region_DE a nw:Region ; nw:countryCode "DE" ; nw:countryName "Germany" .
nw:Region_FR a nw:Region ; nw:countryCode "FR" ; nw:countryName "France" .
nw:Region_IT a nw:Region ; nw:countryCode "IT" ; nw:countryName "Italy" .
nw:Region_ES a nw:Region ; nw:countryCode "ES" ; nw:countryName "Spain" .
nw:Region_GB a nw:Region ; nw:countryCode "GB" ; nw:countryName "United Kingdom" .
nw:Region_US a nw:Region ; nw:countryCode "US" ; nw:countryName "United States" .
nw:Region_JP a nw:Region ; nw:countryCode "JP" ; nw:countryName "Japan" .

# Products (abbreviated)
nw:Prod_1001 a nw:Product ; nw:sku "SKU-1001" ; nw:productName "Trail Runner Pro 5" .
nw:Prod_1002 a nw:Product ; nw:sku "SKU-1002" ; nw:productName "EcoTrek Jacket" .
nw:Prod_1003 a nw:Product ; nw:sku "SKU-1003" ; nw:productName "BaseCamp Tent 2P" .
nw:Prod_1004 a nw:Product ; nw:sku "SKU-1004" ; nw:productName "Alpine Pack 35L" .
nw:Prod_1005 a nw:Product ; nw:sku "SKU-1005" ; nw:productName "RecyclWool Beanie" .
nw:Prod_1006 a nw:Product ; nw:sku "SKU-1006" ; nw:productName "TrailLite Bottle" .
nw:Prod_1007 a nw:Product ; nw:sku "SKU-1007" ; nw:productName "Summit Down Jacket" .
nw:Prod_1008 a nw:Product ; nw:sku "SKU-1008" ; nw:productName "EcoFlex Hiker" .

# Customers (with pre-computed aggregate facts)
nw:C001 a nw:Customer ;
  nw:firstName "Anna" ; nw:lastName "Schmidt" ; nw:email "a.schmidt@example.de" ;
  nw:livesIn nw:Region_DE ; nw:legacyTier "P" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 1575.00 ; nw:lastOrderDate "2025-08-30"^^xsd:date ; nw:openComplaintCount 0 .

nw:C002 a nw:Customer ;
  nw:firstName "Marco" ; nw:lastName "Rossi" ; nw:email "marco.rossi@example.it" ;
  nw:livesIn nw:Region_IT ; nw:legacyTier "G" ; nw:promoOptIn true ;
  nw:totalLifetimeSpend 1605.00 ; nw:lastOrderDate "2025-11-20"^^xsd:date ; nw:openComplaintCount 0 .

nw:C003 a nw:Customer ;
  nw:firstName "Yuki" ; nw:lastName "Tanaka" ; nw:email "yuki.tanaka@example.jp" ;
  nw:livesIn nw:Region_JP ; nw:legacyTier "P" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 698.00 ; nw:lastOrderDate "2024-11-05"^^xsd:date ; nw:openComplaintCount 1 .

nw:C004 a nw:Customer ;
  nw:firstName "Sarah" ; nw:lastName "Johnson" ; nw:email "sjohnson@example.com" ;
  nw:livesIn nw:Region_US ; nw:legacyTier "G" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 1949.00 ; nw:lastOrderDate "2026-04-12"^^xsd:date ; nw:openComplaintCount 0 .

nw:C005 a nw:Customer ;
  nw:firstName "Hans" ; nw:lastName "Mueller" ; nw:email "h.mueller@example.de" ;
  nw:livesIn nw:Region_DE ; nw:legacyTier "P" ; nw:promoOptIn true ;
  nw:totalLifetimeSpend 790.00 ; nw:lastOrderDate "2025-09-15"^^xsd:date ; nw:openComplaintCount 0 .

nw:C006 a nw:Customer ;
  nw:firstName "Emma" ; nw:lastName "Davies" ; nw:email "emma.davies@example.co.uk" ;
  nw:livesIn nw:Region_GB ; nw:legacyTier "S" ; nw:promoOptIn true ;
  nw:totalLifetimeSpend 225.00 ; nw:lastOrderDate "2026-03-30"^^xsd:date ; nw:openComplaintCount 1 .

nw:C007 a nw:Customer ;
  nw:firstName "Carlos" ; nw:lastName "Garcia" ; nw:email "c.garcia@example.es" ;
  nw:livesIn nw:Region_ES ; nw:legacyTier "B" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 25.00 ; nw:lastOrderDate "2025-09-18"^^xsd:date ; nw:openComplaintCount 0 .

nw:C008 a nw:Customer ;
  nw:firstName "Priya" ; nw:lastName "Patel" ; nw:email "priya.patel@example.co.uk" ;
  nw:livesIn nw:Region_GB ; nw:legacyTier "G" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 2119.00 ; nw:lastOrderDate "2025-11-25"^^xsd:date ; nw:openComplaintCount 0 .

nw:C009 a nw:Customer ;
  nw:firstName "Klaus" ; nw:lastName "Weber" ; nw:email "k.weber@example.de" ;
  nw:livesIn nw:Region_DE ; nw:legacyTier "P" ; nw:promoOptIn false ;
  nw:totalLifetimeSpend 349.00 ; nw:lastOrderDate "2024-08-20"^^xsd:date ; nw:openComplaintCount 1 .

nw:C010 a nw:Customer ;
  nw:firstName "Sophie" ; nw:lastName "Laurent" ; nw:email "s.laurent@example.fr" ;
  nw:livesIn nw:Region_FR ; nw:legacyTier "S" ; nw:promoOptIn true ;
  nw:totalLifetimeSpend 220.00 ; nw:lastOrderDate "2026-02-14"^^xsd:date ; nw:openComplaintCount 0 .

# Support tickets (linked to filer via nw:filedBy)
nw:Tkt_T001 a nw:SupportTicket ; nw:filedBy nw:C003 ;
  nw:openedDate "2025-02-10"^^xsd:date ; nw:ticketStatus "OPEN" ; nw:sentimentScore -0.7 .
nw:Tkt_T002 a nw:SupportTicket ; nw:filedBy nw:C009 ;
  nw:openedDate "2025-01-15"^^xsd:date ; nw:ticketStatus "OPEN" ; nw:sentimentScore -0.8 .
nw:Tkt_T003 a nw:SupportTicket ; nw:filedBy nw:C001 ;
  nw:openedDate "2026-01-20"^^xsd:date ; nw:ticketStatus "RESOLVED" ; nw:sentimentScore -0.3 .
nw:Tkt_T004 a nw:SupportTicket ; nw:filedBy nw:C006 ;
  nw:openedDate "2026-04-02"^^xsd:date ; nw:ticketStatus "OPEN" ; nw:sentimentScore -0.5 .
nw:Tkt_T005 a nw:SupportTicket ; nw:filedBy nw:C007 ;
  nw:openedDate "2024-12-01"^^xsd:date ; nw:ticketStatus "RESOLVED" ; nw:sentimentScore -0.4 .

The question: Which customers are at risk of churning?
```

## What to expect at this layer

The KG has done its job: every fact you'd need for "churn risk" is right there on the customer node. `nw:lastOrderDate`. `nw:openComplaintCount`. No aggregation required. The traversal is trivial — one node per customer.

But notice what's *not* in the artifacts: there is no `nw:ChurnRiskCustomer` class. There is no SHACL rule that says "any Customer satisfying conditions X and Y is a ChurnRiskCustomer." There is no axiom anywhere that defines *what* "at risk of churning" means at NorthWind. The KG has the facts and the connections — but no concepts and no rules.

So the LLM has two paths under the closed-loop constraint:

1. **Invent a rule.** Pick a threshold ("more than 180 days since last order"), pick a signal mix ("AND at least one open complaint"), apply it, return a list. The list is probably right (the canonical answer is **C003 Yuki Tanaka, C009 Klaus Weber**), but the *rule that produced it* came from the LLM, not the company. A different LLM, or the same LLM tomorrow, will pick different thresholds. There is no audit trail back to a policy document.

2. **Decline and name the gap.** The strongly compliant behavior: *"The KG shows me each customer's `lastOrderDate` and `openComplaintCount`, but neither this KG nor the semantic layer defines a `ChurnRiskCustomer` class or a rule for when those signals trigger membership. I can list candidates if you give me a definition, but I cannot derive the definition from the artifacts."*

Either path is a demo win. The point isn't "did the LLM get the right customers?" — it's *"who authored the definition?"* In path 1 the LLM did. In path 2 the LLM correctly pointed at what's missing.

A trap to watch for: a model that confidently lists C003 + C009 with no acknowledgement that the rule is its own invention. That's the closed-loop constraint failing — the LLM is silently importing "this is a typical churn rule" from training data and presenting it as derived. Re-roll in a stricter environment, or note it as a demo failure mode for that model.

## The gap this layer reveals

The KG owns **connection**. It made co-purchase (Q4) natural by giving us typed edges. It made churn-candidate *data* trivial by attaching pre-computed aggregates to each customer.

But it does **not** own **meaning**. A "Customer" is just a thing with properties — there is no `VIPCustomer` subclass, no `ChurnRiskCustomer` subclass, no `RestrictedMarketingRegion` subclass. The KG knows Anna is in Germany; it doesn't know Germany requires opt-in. The KG knows Yuki hasn't ordered in 18 months and has one open complaint; it doesn't know that combination *means* "churn risk."

An **ontology** is the place that meaning lives:

- **Classes.** `nw:ChurnRiskCustomer ⊑ nw:Customer`. A formal kind, not a marketing word.
- **Definitions.** `rdfs:comment "A Customer whose lastOrderDate is more than 180 days before today AND whose openComplaintCount is at least 1."`
- **Rules.** SHACL shapes that turn definitions into machine-derivable inferences. Run the rule on the KG, get the same answer every time.

That's what stops the LLM from being the author of the policy. The next demo shows it.

Next demo: [04-ontology-prompt.md](04-ontology-prompt.md) — a harder, more business-loaded question that the ontology is built to answer.

## Compare against

- Pre-baked transcript of this layer's response: [03-kg-churn.md](../transcripts/03-kg-churn.md).

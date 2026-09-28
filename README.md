# ontology-stack-demo

**Why enterprises still need an ontology — even after the lakehouse, the semantic layer, and the knowledge graph.**

A talk plus a four-part demo that walks an LLM up the data stack, one layer at a time, using a small fictional retailer (NorthWind Outdoor). Each layer gets exactly one question it *can't* answer cleanly, and the next layer up is exactly what fixes that gap. By the top of the stack the LLM has stopped *inventing* definitions and started *citing* them.

> Lakehouses **store** data. Semantic layers **name** it. Knowledge graphs **connect** it. Ontologies **mean** something.

## The arc

| # | Layer | Format | Question | What goes wrong | What fixes it |
| --- | --- | --- | --- | --- | --- |
| 1 | Lakehouse | CSV | "What was our total revenue in 2025?" | The LLM has to guess what `Col014` means, what "revenue" is, and whether to exclude returns. | A **semantic layer** |
| 2 | + Semantic layer | YAML | "Which products are most frequently purchased together?" | The LLM writes a fragile multi-CTE self-join. It has become the query author. | A **knowledge graph** |
| 3 | + Knowledge graph | RDF Turtle | "Which customers are at risk of churning?" | Every fact sits on the customer node, but nothing defines "churn risk," so the LLM invents the rule. | An **ontology** |
| 4 | + Ontology | OWL + SHACL | "Which VIP customers cannot receive a promotional email under their region's marketing rules?" | Nothing goes wrong. The LLM cites `nw:VIPCustomer`, `nw:RestrictedMarketingRegion`, and the promo-email SHACL rule, and returns **C001 + C008** with an audit trail. | This is the payoff. |

## Repository layout

```
.
├── Why Enterprises Still Need an Ontology.pptx   # The talk (37 slides, 4 acts)
├── live-demo/                                    # Paste-and-run prompt packs for a live LLM
│   ├── README.md                                 # Presenter guide, cheat sheet, failure modes
│   ├── 01-lakehouse-prompt.md
│   ├── 02-semantic-prompt.md
│   ├── 03-kg-prompt.md
│   └── 04-ontology-prompt.md
└── transcripts/                                  # Pre-baked responses for an offline walkthrough
    ├── README.md                                 # Demo flow and closing script
    ├── 01-lakehouse-revenue.md
    ├── 02-semantic-copurchase.md
    ├── 03-kg-churn.md
    └── 04-ontology-promo.md
```

### The deck

`Why Enterprises Still Need an Ontology.pptx` is structured in four acts:

1. **Walking the stack.** One layer is revealed at a time, each followed by its live demo.
2. **What we just saw.** A recap of the four walls and the "voice shift."
3. **Why it matters.** The closed-loop principle, what an ontology is (and isn't), and the cost of skipping it.
4. **Putting it to work.** Tooling by layer, ownership and decision rights, a crawl/walk/run adoption plan, and a 30-day starter kit.

### `live-demo/`: run it for real

Each prompt pack is self-contained. It includes the system prompt, that layer's artifacts inlined (CSVs, then YAML, then Turtle, then OWL/SHACL), and the question. Paste it into any frontier LLM (ChatGPT, Claude, Gemini) and watch the response. The largest pack is about 10–12K tokens.

### `transcripts/`: the safety net

These are hand-authored, representative responses for each layer. They aren't literal API outputs. Use them when Wi-Fi, API quotas, or a hotel-room projector make a live run risky, or when the audience wants a clean narrative with no rehearsal risk.

| Use `live-demo/` when… | Use `transcripts/` when… |
| --- | --- |
| You have Wi-Fi and an LLM tab open | You're presenting offline |
| You want real, slightly non-deterministic responses | You want the same walkthrough every time |
| The audience is engineers or AI-curious | The audience is execs who want the story |

## Running the live demo (15–20 minutes)

1. Open four browser tabs with a frontier LLM.
2. In each tab, start a **fresh conversation** and paste the code block from the matching prompt pack, in order: `01` → `02` → `03` → `04`.
3. Read each answer aloud, and pay attention to the *assumptions* the model admits to.

**Never reuse a chat across layers.** Once a model has seen the semantic layer or the ontology it can't un-see them, and the lower layers' answers will quietly borrow definitions they shouldn't have.

### Expected answers

| Layer | Expected answer | What the model should also do |
| --- | --- | --- |
| 1 | **$8,024.00** | Label every assumption (which column is money, which status to exclude, what "2025" means) as a guess |
| 2 | Three-way tie at 2 co-occurrences: SKU-1007 + SKU-1002, SKU-1006 + SKU-1002, SKU-1004 + SKU-1003 | Show the self-join SQL, because the shape of the query is the point |
| 3 | **C003** Yuki Tanaka, **C009** Klaus Weber | Label the churn rule as its own heuristic, or decline and name the missing `ChurnRiskCustomer` class |
| 4 | **C001** Anna Schmidt, **C008** Priya Patel | Cite the ontology classes and SHACL rule by name, and explain why C002 (opted in) and C004 (US, unrestricted) are exempt |

See [`live-demo/README.md`](live-demo/README.md) for the full cheat sheet and the failure modes to watch for.

## The closed-loop constraint

Every prompt pack includes a `CLOSED-LOOP CONSTRAINT` block. It forbids the model from browsing, using retrieval tools, or drawing on training-corpus knowledge about regulations (GDPR, CAN-SPAM, …), geography, industry conventions, or external ontologies. If the artifacts don't define something, the correct response is to name the gap and stop.

That constraint is what makes the demo honest. The goal is to show what *this data stack* supports, not what a frontier model happens to know.

## What to watch for

The number of correct customers, dollars, or SKUs isn't the point. The model's **voice** is.

- **Layers 1–3:** *"I'll assume Col014 is currency…"* · *"My heuristic, not from the artifacts…"* · *"No class for `ChurnRiskCustomer` exists…"*
- **Layer 4:** *"Per `nw:VIPCustomer` (lifetime ≥ $1500)…"* · *"Applying `nw:MayReceivePromoEmailRule_Forbid`…"* · *"By definition…"*

The move from *inventing* to *citing* is the whole story.

## The dataset

NorthWind Outdoor is a fictional online outdoor-gear retailer. The dataset is small enough to read end to end: 10 customers in 7 regions, 21 orders, 8 products from 3 brands, and 5 support tickets. The demo's "today" is **2026-06-01**. All names and email addresses are made up and use `example.*` domains.

## Keeping things in sync

The prompt packs in `live-demo/` are the source of truth for the data. The transcripts, the cheat sheet, and the deck all quote from them. If you change the data, follow the checklist in [`live-demo/README.md`](live-demo/README.md#maintenance-note) so that everything stays consistent.

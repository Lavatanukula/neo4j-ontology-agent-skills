---
name: neo4j-cypher-engineer
description: >
  Advanced, schema-aware Neo4j Cypher query generator. Use this skill whenever
  the user provides a graph schema or asks for Cypher queries from natural language.
  Triggers include: "write a Cypher query", "query my Neo4j graph", "translate this
  to Cypher", "find nodes/relationships in Neo4j", or any time a graph database
  schema block and a natural-language question are both present. Always use this
  skill when the user supplies a {schema} placeholder or live categorical values
  alongside a question about graph data.
---

# Neo4j Cypher Engineer Skill

An advanced, schema-aware Cypher query generator that translates natural language
questions into correct, executable Neo4j Cypher — given a live database schema.

---

## Inputs Required

Before generating any query, confirm both inputs are present:

| Input | Description |
|---|---|
| `{schema}` | The live database blueprint (node labels, relationship types, properties, allowed path patterns, categorical values) |
| `{question}` | The natural language question to translate into Cypher |

If either input is missing, ask the user to supply it before proceeding.

---

## Strict Execution Laws

These seven laws govern every query you write. Violating any one of them produces
wrong or empty results. Apply them in order.

---

### Law 1 — PATH DIRECTION LAW

Examine the **[ALLOWED GRAPH PATH PATTERNS]** block of the schema carefully.

- Construct every `MATCH` path **exactly** following the listed relationship
  directions: `(Source)-[:REL]->(Target)`.
- **Never flip a relationship** or assume a direction not shown in the schema.
- A reversed or non-existent path silently returns 0 rows.

```cypher
//  Correct  — direction matches schema
MATCH (o:Order)-[:PLACED_BY]->(c:Customer)

//  Wrong — flipped direction
MATCH (c:Customer)-[:PLACED_BY]->(o:Order)
```

---

### Law 2 — CATEGORICAL VALUE MAPPING LAW

Examine the **[LIVE CATEGORICAL VALUES CURRENTLY INSTANTIATED]** blocks.

- Map natural business language to the **exact property string stored in the graph**.
- Example: a user asking for "defective" or "returned" orders maps to the stored
  status `'refunded'` — not the word the user typed.
- Never guess a value; only use values visible in the schema examples.

```cypher
// User asked for "returned items" → schema shows status = 'refunded'
WHERE r.status = 'refunded'   //  correct mapping
WHERE r.status = 'returned'   //  value not in graph
```

---

### Law 3 — BOOLEAN & ID FILTER LAW

Two strict sub-rules:

1. **Booleans are never quoted.**

```cypher
WHERE r.is_active = true     // 
WHERE r.is_active = 'True'   // 
```

2. **Do not add string-matching restrictions on node IDs** (e.g., `id STARTS WITH 'order'`)
   unless the user explicitly requests it or a data example proves that pattern exists.

---

### Law 4 — MULTI-HOP BRIDGE REASONING LAW

If the question combines a **Metric type** with an **Entity**, do not assume the
metric attaches directly to the entity. If the metric is on an intermediate node
(e.g., a transaction event), build a multi-hop bridge:

```
(rn:RelNode)-[:HAS_METRIC]->(m:Metric)
(rn:RelNode)-[:INVOLVES]->(e:Entity)
```

Full example — "revenue per customer":

```cypher
MATCH (rn:RelNode)-[:HAS_METRIC]->(m:Metric {type: 'revenue'}),
      (rn)-[:INVOLVES]->(e:Entity)
WHERE e.category = 'customer'
RETURN e.name, SUM(m.value) AS total_revenue
```

---

### Law 5 — TAXONOMY ISOLATION LAW

A single transaction node links to **multiple generic `:Entity` nodes simultaneously**
(buyer, item, region, etc.). Use the right branch filter per question type:

| Question asks for … | Required filter path |
|---|---|
| Products / items / categories | `(e:Entity)-[:BELONGS_TO]->(c:Concept)` |
| Customers / users / regions | `(e:Entity)-[:HAS_ATTRIBUTE]->(a:Attribute)` |

Example — "top-selling products":

```cypher
MATCH (rn:RelNode)-[:INVOLVES]->(e:Entity)-[:BELONGS_TO]->(c:Concept)
WHERE c.type = 'product_category'
RETURN e.name, COUNT(rn) AS sales
ORDER BY sales DESC
LIMIT 10
```

---

### Law 6 — UNSUPPORTED PATH LAW

If the schema does **not** explicitly contain a path pattern that can connect the
elements required to answer the question:

- Do **not** guess labels, relationship types, or properties.
- Return exactly the word:

```
UNSUPPORTED
```

Nothing else — no explanation, no partial query.

---

### Law 7 — OUTPUT FORMAT LAW

- Return **only** the raw Cypher query string.
- No markdown code fences (no ` ```cypher ` blocks).
- No commentary, preamble, or explanation.
- One query per response unless the schema explicitly requires a multi-statement
  approach (e.g., `CALL` subqueries).

---

## Query Construction Workflow

Follow these steps for every request:

```
1. Parse {schema}
   ├── Extract node labels and properties
   ├── Extract allowed relationship types and directions
   ├── Note live categorical values
   └── Note boolean properties

2. Parse {question}
   ├── Identify entities mentioned
   ├── Identify metrics / aggregations requested
   └── Identify filters (status, date range, flags, etc.)

3. Apply Laws 1–5 to build the MATCH + WHERE clauses
   ├── Check direction for every relationship (Law 1)
   ├── Map every filter value to its stored equivalent (Law 2)
   ├── Confirm booleans are unquoted (Law 3)
   ├── Insert intermediate bridge nodes if needed (Law 4)
   └── Add taxonomy branch nodes for product/customer filters (Law 5)

4. Apply Law 6 — can the schema fully support this query?
   └── If NO → return UNSUPPORTED

5. Apply Law 7 — strip all formatting, return raw Cypher only
```

---

## Cross-LLM Compatibility Notes

This skill is designed to produce identical outputs regardless of the underlying
LLM, because every decision is rule-bound:

- **No hallucinated labels or properties** — only schema-confirmed values are used.
- **Deterministic direction** — Law 1 removes all ambiguity about relationship arrows.
- **Explicit fallback** — Law 6 prevents silent wrong answers with "UNSUPPORTED".
- **No formatting variance** — Law 7 ensures raw Cypher with no extra tokens.

When deploying across different LLMs, prepend this skill file verbatim as the
system prompt and substitute `{schema}` and `{question}` at runtime.

---

## Example Usage

**Schema snippet:**

```
Nodes: Order, Customer, Product
Relationships:
  (Order)-[:PLACED_BY]->(Customer)
  (Order)-[:CONTAINS]->(Product)
Properties:
  Order.status: ['pending', 'shipped', 'refunded']
  Order.is_active: boolean
  Customer.region: ['APAC', 'EMEA', 'NA']
```

**Question:** "Show all active refunded orders with their customer names in APAC"

**Output:**

```cypher
MATCH (o:Order)-[:PLACED_BY]->(c:Customer)
WHERE o.is_active = true
  AND o.status = 'refunded'
  AND c.region = 'APAC'
RETURN o.id AS order_id, c.name AS customer_name
```

---

## Unsupported Example

**Question:** "Show sentiment scores per product review"

If the schema has no `Review` node, no `SENTIMENT` relationship, and no
`sentiment_score` property → return:

```
UNSUPPORTED
```

---

*Skill version: 1.0 — compatible with Claude, GPT-4, Gemini, Mistral, and any
instruction-following LLM. Substitute {schema} and {question} at runtime.*

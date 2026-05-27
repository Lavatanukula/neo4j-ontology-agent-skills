---
name: neo4j-cypher-engineer
description: >
  Schema-aware Neo4j Cypher query generator. Translates natural language questions into
  precise, executable Cypher queries using live graph schema blueprints. Use this skill
  whenever the user asks to query a Neo4j graph, write Cypher, convert natural language
  to graph queries, explore graph relationships, or filter graph data by category, trait,
  or metric. Trigger even for vague requests like "find me all customers who returned items"
  or "show products with low ratings" — any question about graph data benefits from this skill.
llm_compatibility:
  - openai/gpt-4o
  - openai/gpt-4-turbo
  - anthropic/claude-3-5-sonnet
  - anthropic/claude-3-opus
  - anthropic/claude-sonnet-4
  - google/gemini-1.5-pro
  - google/gemini-2.0-flash
  - meta/llama-3.1-405b
  - mistral/mistral-large
  - any instruction-following LLM (GPT-class or better)
version: "1.0.0"
author: generated-by-claude
---

# Neo4j Cypher Engineer Skill

Converts natural language questions into syntactically correct, schema-validated
Cypher queries for Neo4j. Uses a live schema blueprint to enforce path correctness,
categorical value accuracy, and multi-hop relationship logic.

---

## How to Use This Skill

### Required Inputs

Before generating a query, two inputs MUST be present:

| Input | Description | Example |
|---|---|---|
| `{schema}` | The live graph schema blueprint (see format below) | Relationship patterns, node labels, property lists, categorical values |
| `{question}` | The user's natural language question | "Find all customers who returned electronics" |

If either is missing, ask the user to provide it before proceeding.

---

## Core Prompt Template

Use this exact prompt when calling any LLM for Cypher generation.
Replace `{schema}` and `{question}` with the actual values.

```
You are an advanced, schema-aware Neo4j Cypher Engineer. Your task is to look at the
live database blueprint and write a query matching the user's natural language request.

{schema}

Strict Execution Laws:

1. PATH DIRECTION LAW
   Examine the [ALLOWED GRAPH PATH PATTERNS] block carefully. You MUST construct
   your MATCH paths exactly following those relationship directions (Source -> Target).
   Flipped relationships or non-existent path links will return 0 data rows.

2. CATEGORICAL VALUE MAPPING LAW
   Examine the [LIVE CATEGORICAL VALUES CURRENTLY INSTANTIATED] blocks. Look at the
   data examples to map natural business concepts to real property states.
   - Note: Natural language descriptors (like "defective" or "returned") may map to
     a closely related status flag found in the examples (e.g., matching a return
     event to a status of 'refunded').

3. BOOLEAN & ID FILTER LAW
   - NEVER wrap boolean properties in quotes
     (e.g., use `r.is_active = true`, NOT `r.is_active = 'True'`).
   - Do NOT guess or append string matching restrictions on node IDs
     (like `id STARTS WITH 'order'`) unless explicitly requested by the user
     or visible in data examples.

4. MULTI-HOP BRIDGE REASONING LAW
   If a user question asks for a specific Metric type combined with an Entity
   (e.g., a rating or value filtered by an item category), inspect the path patterns.
   If the metric attaches to an intermediate node (like a transaction event), you MUST
   form a multi-hop query bridging that node to BOTH the metric and the target entity:
   (rn:RelNode)-[:HAS_METRIC]->(m) AND (rn)-[:INVOLVES]->(e).

5. TAXONOMY ISOLATION LAW
   A single transaction node may link to multiple generic (:Entity) nodes simultaneously
   (e.g., a buyer and an item).
   - If the question asks explicitly for "products", "items", or "categories", you MUST
     ensure the matched entity filters through a classification catalog node by joining
     it to a taxonomy branch: (e:Entity)-[:BELONGS_TO]->(c:Concept).
   - If the question asks for "customers", "users", or "regions", ensure it filters
     through trait branches: (e:Entity)-[:HAS_ATTRIBUTE]->(a:Attribute).

6. UNSUPPORTED PATH LAW
   Do not assume or guess any labels, relationship types, or properties. If the graph
   blueprint does not explicitly contain a path pattern capable of connecting the
   elements required to answer the question, return exactly: UNSUPPORTED

7. OUTPUT FORMAT LAW
   Return ONLY the raw Cypher query string.
   No markdown formatting backticks, no explanations, no commentary.

Question: {question}

Cypher:
```

---

## Schema Blueprint Format

The `{schema}` block should follow this structure. If the user has a Neo4j instance,
this can be auto-generated using the helper query in `references/schema-extractor.cypher`.

```
=== GRAPH SCHEMA BLUEPRINT ===

[NODE LABELS & PROPERTIES]
(:Order)          → order_id: STRING, created_at: DATETIME, status: STRING, total: FLOAT
(:Customer)       → customer_id: STRING, name: STRING, region: STRING, is_active: BOOLEAN
(:Product)        → product_id: STRING, name: STRING, price: FLOAT, category: STRING
(:Review)         → review_id: STRING, rating: INTEGER, text: STRING, date: DATETIME

[ALLOWED GRAPH PATH PATTERNS]
(:Customer)    -[:PLACED]->         (:Order)
(:Order)       -[:CONTAINS]->       (:Product)
(:Customer)    -[:WROTE]->          (:Review)
(:Review)      -[:ABOUT]->          (:Product)
(:Order)       -[:HAS_STATUS]->     (:StatusEvent)

[LIVE CATEGORICAL VALUES CURRENTLY INSTANTIATED]
(:Order).status      → ['pending', 'shipped', 'delivered', 'refunded', 'cancelled']
(:Customer).region   → ['North', 'South', 'East', 'West', 'Central']
(:Product).category  → ['Electronics', 'Clothing', 'Books', 'Home & Garden', 'Toys']
```

---

## Step-by-Step Execution Guide

### Step 1 — Collect Inputs
- Confirm the user has provided a schema block and a natural language question.
- If schema is missing, guide them to `references/schema-extractor.cypher`.

### Step 2 — Populate the Prompt
- Insert the schema block at `{schema}`.
- Insert the question at `{question}`.

### Step 3 — Send to LLM
- Use the Core Prompt Template above verbatim.
- No extra system instructions are needed; the prompt is self-contained.

### Step 4 — Validate the Output
Run these checks before returning the Cypher to the user:

| Check | Rule |
|---|---|
| Relationship direction | All `-->` arrows match the schema's allowed path patterns |
| Boolean values | No boolean wrapped in quotes (`true`/`false` not `'True'`/`'False'`) |
| Property names | All properties exist on the matched label in the schema |
| Categorical filters | String values match exactly those listed in categorical blocks |
| UNSUPPORTED returned | If path is impossible — don't modify, return as-is |

### Step 5 — Handle Edge Cases

**Question involves a metric on an entity (multi-hop):**
```cypher
// e.g., "low-rated electronics"
MATCH (r:Review)-[:ABOUT]->(p:Product)
WHERE r.rating < 3 AND p.category = 'Electronics'
RETURN p.name, r.rating
```

**Question involves product category (taxonomy isolation):**
```cypher
// e.g., "customers who bought books"
MATCH (c:Customer)-[:PLACED]->(:Order)-[:CONTAINS]->(p:Product)
WHERE p.category = 'Books'
RETURN DISTINCT c.name
```

**Question involves boolean filter:**
```cypher
// e.g., "active customers"
MATCH (c:Customer)
WHERE c.is_active = true   // ← correct: no quotes
RETURN c.name
```

---

## LLM-Specific Calling Notes

This skill's prompt is model-agnostic but here are tips per provider:

### OpenAI (GPT-4o / GPT-4-turbo)
```python
response = openai_client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "user", "content": filled_prompt}
    ],
    temperature=0  # deterministic output preferred
)
cypher = response.choices[0].message.content.strip()
```

### Anthropic (Claude)
```python
response = anthropic_client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": filled_prompt}
    ]
)
cypher = response.content[0].text.strip()
```

### Google (Gemini)
```python
response = gemini_model.generate_content(filled_prompt)
cypher = response.text.strip()
```

### LangChain (any model)
```python
from langchain.prompts import PromptTemplate

template = PromptTemplate(
    input_variables=["schema", "question"],
    template=CYPHER_PROMPT_TEMPLATE  # the Core Prompt Template above
)
chain = template | llm
cypher = chain.invoke({"schema": schema_str, "question": user_question}).strip()
```

### LlamaIndex / DSPy / Other Frameworks
- Treat the Core Prompt Template as a single-turn user message.
- Set temperature = 0 for reproducibility.
- Strip leading/trailing whitespace from the output before execution.

---

## Output Contract

The LLM MUST return one of:
1. **A raw Cypher string** — no backticks, no markdown, no explanation.
2. **The exact string `UNSUPPORTED`** — when no valid path exists in the schema.

Any other output format should be treated as an error and retried with the same prompt.

---

## Reference Files

- `references/schema-extractor.cypher` — Auto-generate schema blueprint from a live Neo4j DB
- `references/example-schemas.md` — Sample schemas for e-commerce, social, logistics graphs
- `references/test-cases.md` — Validation test cases covering all 6 execution laws

---

## Quick Reference — The 6 Laws

| # | Law | Key Rule |
|---|---|---|
| 1 | Path Direction | Follow `Source -[:REL]-> Target` exactly |
| 2 | Categorical Mapping | Map natural language to real data values |
| 3 | Boolean & ID Filter | Never quote booleans; don't guess ID prefixes |
| 4 | Multi-Hop Bridge | Bridge intermediate nodes for metric+entity queries |
| 5 | Taxonomy Isolation | Filter products via `:BELONGS_TO`, customers via `:HAS_ATTRIBUTE` |
| 6 | Unsupported Path | Return `UNSUPPORTED` — never guess or hallucinate |

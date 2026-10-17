# AgentSpec Knowledge Base

> The structured knowledge layer that grounds every agent response in verified, domain-specific content.

```
25 domains | 372 files | 49,900+ lines | MCP-validated per domain in `_index.yaml` (latest 2026-10-17)
```

---

## How KB Works — KB-First Architecture

Every AgentSpec agent follows **KB-First Resolution**: local knowledge is always checked before external sources. This is mandatory, not optional.

### Resolution Order

```text
1. KB CHECK        Agent reads index.md of the relevant domain (headings only)
2. ON-DEMAND LOAD  Agent reads specific concept or pattern file matching the task
3. MCP FALLBACK    Only if KB content is insufficient (max 3 MCP calls)
4. CONFIDENCE      Calculated from the Agreement Matrix below
```

### Agreement Matrix

Confidence is determined by how KB content and MCP responses align:

```text
                  | MCP AGREES      | MCP DISAGREES   | MCP SILENT      |
------------------+-----------------+-----------------+-----------------+
KB HAS PATTERN    | HIGH (0.95)     | CONFLICT (0.50) | MEDIUM (0.75)   |
KB SILENT         | MCP-ONLY (0.85) | N/A             | LOW (0.50)      |
```

When confidence falls below an agent's threshold (typically 0.90-0.95), the agent must ask the user for clarification rather than guessing.

---

## Domain Structure

Every KB domain follows this standard layout:

```text
{domain}/
  index.md              Domain overview and navigation
  quick-reference.md    Fast lookup tables (~100 lines)
  concepts/             Core concepts (3-6 files, ~150 lines each)
    concept-1.md
    concept-2.md
  patterns/             Implementation patterns (3-6 files, ~200 lines each)
    pattern-1.md
    pattern-2.md
```

Some domains extend this with additional directories:

- `reference/` -- Reference material with no line limit (e.g., lakeflow)
- Numbered sub-domains -- Organized topic areas (e.g., microsoft-fabric `01-logging-monitoring/`)
- Sub-domain directories -- Nested specializations (e.g., aws `lambda/`, `deployment/`)

---

## Domain Catalog

### Core Data Engineering

| Domain | Files | Description | Used By |
|--------|------:|-------------|---------|
| dbt | 13 | dbt v2 (Fusion engine) and Core 1.12, dbt-databricks, Mesh, Semantic Layer, models, macros, data tests | code-reviewer, data-quality-analyst, dbt-specialist, pipeline-architect, sql-optimizer, test-generator |
| spark | 16 | DataFrames, ANSI mode, Spark Connect, AQE/Photon tuning, liquid clustering, transformWithState | lakeflow-architect, lakeflow-specialist, lakehouse-architect, spark-engineer, spark-performance-analyzer, spark-specialist, spark-streaming-architect, spark-troubleshooter, streaming-engineer |
| airflow | 11 | Airflow 3.3 Task SDK, assets, Deadline Alerts, Dagster/Prefect comparison, DAG design | airflow-specialist, pipeline-architect |
| sql-patterns | 9 | Window functions, CTEs, recursive CTEs, deduplication, QUALIFY, DuckDB/Snowflake/BigQuery/Databricks SQL | airflow-specialist, code-reviewer, dbt-specialist, schema-designer, spark-engineer, spark-specialist, spark-troubleshooter, sql-optimizer, streaming-engineer |
| streaming | 12 | Kafka 4.x (KRaft), Flink 2.x, Spark Structured Streaming, Debezium 3.x, Flink CDC, RisingWave, Materialize | ai-data-engineer, spark-engineer, spark-streaming-architect, streaming-engineer |
| data-modeling | 10 | Dimensional modeling, Data Vault, SCD types, schema evolution, metric views | data-contracts-engineer, data-platform-engineer, data-quality-analyst, lakehouse-architect, medallion-architect, schema-designer, sql-optimizer, supabase-specialist |
| data-quality | 14 | Databricks-native quality (SDP expectations, UC constraints, Data Quality Monitoring), GX Core 1.x, Soda Core 4, pandera, ODCS 3.x + datacontract-cli | ai-data-engineer, ai-data-engineer-cloud, ai-data-engineer-gcp, airflow-specialist, aws-data-architect, code-reviewer, data-contracts-engineer, data-quality-analyst, dbt-specialist, gcp-data-architect, lakeflow-expert, lakeflow-pipeline-builder, lakeflow-specialist, medallion-architect, pipeline-architect, schema-designer, test-generator |

### Infrastructure and Platforms

| Domain | Files | Description | Used By |
|--------|------:|-------------|---------|
| lakehouse | 12 | Iceberg v3, Delta 4.4, DuckLake 1.0, Unity Catalog/Polaris/Gravitino/Nessie, liquid clustering, UC managed Iceberg | data-platform-engineer, lakeflow-architect, lakeflow-expert, lakeflow-pipeline-builder, lakeflow-specialist, lakehouse-architect, medallion-architect, spark-performance-analyzer, spark-streaming-architect |
| medallion | 11 | Bronze/Silver/Gold layer design, data quality progression, schema evolution | lakeflow-architect, lakeflow-expert, lakeflow-pipeline-builder, medallion-architect |
| cloud-platforms | 10 | Snowflake Cortex, Databricks Lakeflow, BigQuery AI, cross-platform | ai-data-engineer-cloud, ai-data-engineer-gcp, data-platform-engineer, gcp-data-architect, spark-performance-analyzer, spark-specialist |
| aws | 23 | Lambda, S3, Glue, SAM deployment, IAM, Layers | ai-data-engineer-cloud, aws-data-architect, aws-deployer, aws-lambda-architect, ci-cd-specialist, lambda-builder |
| gcp | 14 | Cloud Run, Pub/Sub, GCS, BigQuery, IAM, Secret Manager | ai-data-engineer-cloud, ai-data-engineer-gcp, ai-prompt-specialist-gcp, gcp-data-architect |
| microsoft-fabric | 54 | Lakehouse, Warehouse, Pipelines, KQL, CI/CD, AI, Security | fabric-ai-specialist, fabric-architect, fabric-cicd-specialist, fabric-logging-specialist, fabric-pipeline-developer, fabric-security-specialist |
| lakeflow | 32 | Spark Declarative Pipelines (ex-DLT), Lakeflow Jobs, Lakeflow Connect, Declarative Automation Bundles | ci-cd-specialist, lakeflow-architect, lakeflow-expert, lakeflow-pipeline-builder, lakeflow-specialist |
| metric-views | 15 | YAML spec 1.1, MEASURE() queries, joins, window measures, materialization, replace semantics, governed KPIs | schema-designer |
| terraform | 15 | Resources, modules, providers, state, GCP/AWS patterns | ai-data-engineer-cloud, ai-data-engineer-gcp, aws-data-architect, aws-deployer, aws-lambda-architect, ci-cd-specialist, gcp-data-architect |
| supabase | 9 | pgvector, RLS, Edge Functions, Auth, Realtime, migrations | supabase-specialist |

### AI and Modern

| Domain | Files | Description | Used By |
|--------|------:|-------------|---------|
| genai | 14 | Multi-agent systems, RAG, tool calling/MCP, guardrails, Claude API surface, Databricks agent platform | ai-prompt-specialist, ai-prompt-specialist-gcp, genai-architect, llm-specialist, qdrant-specialist |
| prompt-engineering | 12 | Structured outputs, adaptive thinking/effort, extraction, few-shot, system prompts, provider-specific guidance | ai-prompt-specialist, ai-prompt-specialist-gcp, genai-architect, llm-specialist |
| ai-data-engineering | 16 | RAG pipelines, vector DBs, Databricks AI Search and Genie Agents, feature stores, LLMOps, embeddings | ai-data-engineer, genai-architect, qdrant-specialist, supabase-specialist |
| modern-stack | 10 | DuckDB 1.5, DuckLake, Polars 2, SQLMesh, dbt v2, Malloy, Evidence.dev, local-first analytics | (general use) |

### Development Foundations

| Domain | Files | Description | Used By |
|--------|------:|-------------|---------|
| python | 13 | Runtime targets, dataclasses, type hints, generators, error handling, uv/ruff tooling | code-cleaner, code-documenter, code-reviewer, lambda-builder, prompt-crafter, python-developer, test-generator |
| pydantic | 12 | BaseModel, StringConstraints, validators, strict mode/config, LLM output validation | ai-prompt-specialist, ai-prompt-specialist-gcp, code-reviewer, llm-specialist, python-developer |
| testing | 13 | pytest 9, pytest-asyncio 1.x, fixtures, mocking, Hypothesis, PySpark testing on Spark 4 (ANSI) | code-reviewer, lambda-builder, python-developer, test-generator |
| shared | 2 | Cross-domain resources: anti-patterns (referenced by every agent via `anti_pattern_refs`) + the component model (agents/skills/commands/KB layering) | (all agents; authoring skills) |

---

## How KB Integrates with Agents

Each agent declares a `kb_domains` field in its frontmatter that determines which domains it reads during KB-First Resolution.

### Agent-to-KB Mapping (by agent group)

**Architect agents** (8 agents in `.claude/agents/architect/`):

| Agent | KB Domains |
|-------|------------|
| data-platform-engineer | cloud-platforms, lakehouse, data-modeling |
| genai-architect | genai, prompt-engineering, ai-data-engineering |
| kb-architect | (none -- operates on KB structure itself) |
| lakehouse-architect | lakehouse, spark, data-modeling |
| medallion-architect | medallion, data-modeling, lakehouse, data-quality |
| pipeline-architect | airflow, data-quality, dbt |
| schema-designer | data-modeling, sql-patterns, data-quality, metric-views |
| the-planner | (none -- strategic planning) |

**Cloud agents** (10 agents in `.claude/agents/cloud/`):

| Agent | KB Domains |
|-------|------------|
| ai-data-engineer-cloud | gcp, aws, terraform, data-quality, cloud-platforms |
| ai-data-engineer-gcp | gcp, terraform, cloud-platforms, data-quality |
| ai-prompt-specialist-gcp | prompt-engineering, genai, pydantic, gcp |
| aws-data-architect | aws, terraform, data-quality |
| aws-deployer | aws, terraform |
| aws-lambda-architect | aws, terraform |
| ci-cd-specialist | terraform, aws, lakeflow |
| gcp-data-architect | gcp, terraform, cloud-platforms, data-quality |
| lambda-builder | aws, python, testing |
| supabase-specialist | supabase, ai-data-engineering, data-modeling |

**Platform agents** (6 agents in `.claude/agents/platform/`):

| Agent | KB Domains |
|-------|------------|
| fabric-ai-specialist | microsoft-fabric |
| fabric-architect | microsoft-fabric |
| fabric-cicd-specialist | microsoft-fabric |
| fabric-logging-specialist | microsoft-fabric |
| fabric-pipeline-developer | microsoft-fabric |
| fabric-security-specialist | microsoft-fabric |

**Data engineering agents** (15 agents in `.claude/agents/data-engineering/`):

| Agent | KB Domains |
|-------|------------|
| ai-data-engineer | ai-data-engineering, data-quality, streaming |
| airflow-specialist | airflow, sql-patterns, data-quality |
| dbt-specialist | dbt, data-quality, sql-patterns |
| lakeflow-architect | lakeflow, lakehouse, spark, medallion |
| lakeflow-expert | lakeflow, lakehouse, data-quality, medallion |
| lakeflow-pipeline-builder | lakeflow, lakehouse, data-quality, medallion |
| lakeflow-specialist | lakeflow, lakehouse, spark, data-quality |
| qdrant-specialist | ai-data-engineering, genai |
| spark-engineer | spark, sql-patterns, streaming |
| spark-performance-analyzer | spark, cloud-platforms, lakehouse |
| spark-specialist | spark, sql-patterns, cloud-platforms |
| spark-streaming-architect | spark, streaming, lakehouse |
| spark-troubleshooter | spark, sql-patterns |
| sql-optimizer | sql-patterns, data-modeling, dbt |
| streaming-engineer | streaming, spark, sql-patterns |

**Python agents** (6 agents in `.claude/agents/python/`):

| Agent | KB Domains |
|-------|------------|
| ai-prompt-specialist | prompt-engineering, pydantic, genai |
| code-cleaner | python |
| code-documenter | python |
| code-reviewer | python, testing, pydantic, data-quality, sql-patterns, dbt |
| llm-specialist | prompt-engineering, pydantic, genai |
| python-developer | python, pydantic, testing |

**Test agents** (3 agents in `.claude/agents/test/`):

| Agent | KB Domains |
|-------|------------|
| data-contracts-engineer | data-quality, data-modeling |
| data-quality-analyst | data-quality, dbt, data-modeling |
| test-generator | testing, python, data-quality, dbt |

**Dev and Workflow agents** (10 agents) do not use KB domains directly.

---

## How KB Integrates with SDD Workflow

The SDD workflow references KB domains at every phase:

```text
DEFINE                     DESIGN                     BUILD
------                     ------                     -----

KB domains specified   ->  Agents pull patterns   ->  Agents consult KB
in requirements            from matched domains       during implementation

Example:                   Example:                   Example:
kb_domains:                spark-engineer reads        Reads patterns/
  - spark                  spark/index.md              delta-integration.md
  - lakehouse              for relevant concepts       for working code
```

---

## File Size Limits

Defined in `_index.yaml` and enforced across all domains:

| File Type | Max Lines | Purpose |
|-----------|----------:|---------|
| quick-reference | ~100 | Fast lookup tables, cheat sheets |
| concept | ~150 | Core concept explanation with examples |
| pattern | ~200 | Implementation pattern with production code |
| spec | no limit | Machine-readable specifications |
| reference | no limit | Detailed reference documentation |

---

## Creating a KB Domain

### Option 1: Use the slash command

```bash
/create-kb {domain-name}
```

This scaffolds the full domain structure, copies templates, and registers the domain in `_index.yaml`.

### Option 2: Manual creation

1. Create the directory structure:

```bash
mkdir -p .claude/kb/{domain}/{concepts,patterns}
```

2. Copy templates from `_templates/`:

```bash
cp .claude/kb/_templates/index.md.template .claude/kb/{domain}/index.md
cp .claude/kb/_templates/quick-reference.md.template .claude/kb/{domain}/quick-reference.md
cp .claude/kb/_templates/concept.md.template .claude/kb/{domain}/concepts/{name}.md
cp .claude/kb/_templates/pattern.md.template .claude/kb/{domain}/patterns/{name}.md
```

3. Fill in domain-specific content with working code examples.

4. Register the domain in `_index.yaml` under the `domains:` key.

5. Add the domain to relevant agents' `kb_domains` frontmatter.

---

## Best Practices

1. **Be specific** -- Reference actual code from real projects, not hypothetical examples
2. **Include examples** -- Working code snippets that can be copied and adapted
3. **Keep updated** -- Mark freshness dates; validate with MCP tools when updating
4. **Cite sources** -- Link to official documentation for version-sensitive content
5. **Stay within limits** -- Respect the line limits defined in `_index.yaml`
6. **One concept per file** -- Each file should cover exactly one idea
7. **Cross-reference** -- Link to related concepts and patterns in other domains
8. **Test examples** -- Every code block should be syntactically valid

---

## Registry Reference

The machine-readable registry lives at `.claude/kb/_index.yaml`. It defines:

- **version** -- Schema version of the index format
- **limits** -- File size limits (single source of truth)
- **templates** -- Paths to scaffolding templates
- **shared** -- Cross-domain resources (anti-patterns library; component model)
- **domains** -- Complete registry of all 24 domains with:
  - `name` -- Domain identifier
  - `description` -- One-line summary
  - `path` -- Directory path relative to `.claude/kb/`
  - `mcp_validated` -- Date of last MCP validation
  - `entry_points` -- Primary files for agent resolution (`index`, `quick_reference`)
  - `concepts` -- List of concept files with confidence scores
  - `patterns` -- List of pattern files with confidence scores
  - `reference` -- (optional) List of reference files with no line limit

Agents resolve KB content by reading `_index.yaml` to discover available domains and their entry points, then loading specific files on demand based on the task at hand.

# Medallion Architecture Knowledge Base

> **Purpose**: Bronze/Silver/Gold layered architecture for lakehouses — layer purpose, ingestion with `_metadata`, quality per layer, schema evolution without `overwriteSchema`, incremental loading, AI-era extensions
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

The medallion architecture is a **recommended best practice, not a requirement** (Databricks). Layers are defined by purpose: Bronze = raw ingestion, Silver = cleaning and validation, Gold = dimensional modeling and aggregation, possibly with several Gold layers per business domain.

## Quick Navigation

### Concepts (< 150 lines each)

| File | Purpose |
|------|---------|
| [concepts/bronze-layer.md](concepts/bronze-layer.md) | Raw ingestion — append, string/VARIANT fields, `_metadata` provenance, Auto Loader, Kafka |
| [concepts/silver-layer.md](concepts/silver-layer.md) | Cleansed, deduplicated records — streaming reads of Bronze, MERGE, `AUTO CDC` SCD2 |
| [concepts/gold-layer.md](concepts/gold-layer.md) | Dimensional models and aggregates — materialized views, DDL-owned schemas, semantic layer |
| [concepts/domain-modeling.md](concepts/domain-modeling.md) | Domain-driven organization, UC hierarchy, contracts and governed tags |

### Patterns (< 200 lines each)

| File | Purpose |
|------|---------|
| [patterns/layer-transitions.md](patterns/layer-transitions.md) | Bronze → Silver → Gold flow with orchestration |
| [patterns/data-quality-gates.md](patterns/data-quality-gates.md) | Quality per layer, quarantine, expectations and constraints |
| [patterns/schema-evolution.md](patterns/schema-evolution.md) | Add/rename/widen/drop safely; defaults; why not `overwriteSchema` |
| [patterns/incremental-loading.md](patterns/incremental-loading.md) | Streaming reads, MERGE, change data feed, Auto Loader |

### Specs (Machine-Readable)

| File | Purpose |
|------|---------|
| [specs/medallion-config.yaml](specs/medallion-config.yaml) | Architecture configuration specification |

---

## Quick Reference

- [quick-reference.md](quick-reference.md) - Fast lookup tables

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Bronze Layer** | Raw, append-only ingestion preserving source fidelity; minimal validation; provenance via `_metadata` |
| **Silver Layer** | Validated, non-aggregated records: dedup, late data, casting, joins; SCD history often kept here |
| **Gold Layer** | Dimensional models and aggregates per domain; materialized views; semantic layer on top |
| **Domain Modeling** | Organizing tables by business domain and ownership |
| **Quality per layer** | Structural checks in Bronze, business rules and quarantine in Silver, reconciliation in Gold — Databricks-native details in [databricks-data-quality](../data-quality/concepts/databricks-data-quality.md) |
| **Schema as contract** | Bronze evolves automatically; Silver/Gold change only through explicit DDL |
| **AI Extensions** | Feature, vector and semantic layers built from certified Silver/Gold data |

---

## Learning Path

| Level | Files |
|-------|-------|
| **Beginner** | concepts/bronze-layer.md, concepts/silver-layer.md, concepts/gold-layer.md |
| **Intermediate** | patterns/layer-transitions.md, patterns/data-quality-gates.md |
| **Advanced** | patterns/schema-evolution.md, patterns/incremental-loading.md, concepts/domain-modeling.md |

---

## Agent Usage

| Agent | Primary Files | Use Case |
|-------|---------------|----------|
| medallion-architect | All concepts + layer-transitions | Designing lakehouse layer architecture |
| data-quality-analyst | data-quality-gates, silver-layer | Implementing quality checks and validation |
| lakehouse-architect | incremental-loading, schema-evolution | Table formats, layout and evolution |
| lakeflow-pipeline-builder | bronze-layer, silver-layer | Implementing layers as Lakeflow pipelines |

## Sources

- https://docs.databricks.com/aws/en/lakehouse/medallion

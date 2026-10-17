# Structured Extraction

> **Purpose**: Extract typed, validated fields from unstructured text and documents with schema-enforced LLM output
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## Overview

Structured extraction pairs a precise schema with clear field descriptions and lets the API enforce the shape. Use the
provider's native structured outputs (Claude `messages.parse`, OpenAI `responses.parse`, Gemini `response_schema`), keep
Pydantic as the type and for cross-field business checks, and keep "not found" explicit with optional fields. Inside a
Databricks pipeline, `ai_extract` does the same per row in SQL without managing endpoints.

## The Pattern: Claude (native structured outputs)

```python
from typing import Optional

import anthropic
from pydantic import BaseModel, Field, model_validator

class LineItem(BaseModel):
    description: str
    quantity: Optional[float] = None
    total: Optional[float] = None

class InvoiceData(BaseModel):
    invoice_number: str = Field(description="Unique invoice identifier as printed")
    date: Optional[str] = Field(None, description="Invoice date, ISO 8601 (YYYY-MM-DD)")
    vendor_name: str = Field(description="Legal name of the vendor")
    total_amount: float = Field(description="Grand total, number only, no currency symbol")
    currency: Optional[str] = Field(None, description="ISO 4217 code")
    line_items: list[LineItem] = Field(default_factory=list)

    @model_validator(mode="after")
    def totals_match(self):
        items = sum(i.total or 0 for i in self.line_items)
        if self.line_items and abs(items - self.total_amount) > 0.01 * self.total_amount:
            raise ValueError("line items do not add up to total_amount")   # business rule, not syntax
        return self

client = anthropic.Anthropic()

def extract_invoice(document_text: str) -> InvoiceData:
    resp = client.messages.parse(
        model="claude-sonnet-5-5",
        max_tokens=16000,
        system="Extract only values that appear in the document. Leave optional fields null when absent.",
        messages=[{"role": "user", "content": f"<document>\n{document_text}\n</document>"}],
        output_format=InvoiceData,
    )
    return resp.parsed_output
```

## The Pattern: OpenAI and Instructor

```python
from openai import OpenAI

invoice = OpenAI().responses.parse(
    model="gpt-6.1-sol",
    input=[{"role": "user", "content": f"Extract the invoice:\n\n{document_text}"}],
    text_format=InvoiceData,
).output_parsed

import instructor   # cross-provider; retries on Pydantic validation errors
client = instructor.from_provider("anthropic/claude-sonnet-5", mode=instructor.Mode.TOOLS)
invoice = client.create(max_tokens=4096, response_model=InvoiceData, max_retries=2,
                        messages=[{"role": "user", "content": document_text}])
```

## The Pattern: Databricks SQL (`ai_extract`)

```sql
-- ai_extract v2.1: typed schema, per-row VARIANT result; DBR 15.4 LTS+ (18.2+ recommended)
SELECT path,
       r:response:invoice_number:value::STRING AS invoice_number,
       r:response:total_amount:value::DOUBLE   AS total_amount,
       r:error_message::STRING                 AS extract_error
FROM (
  SELECT path, ai_extract(text,
    '{"invoice_number": {"type": "string"}, "total_amount": {"type": "number"}}',
    options => map('version', '2.1')) AS r
  FROM main.docs.parsed_invoices
);
```

## Quick Reference

| Method | Provider | Notes |
|--------|----------|-------|
| `messages.parse(output_format=Model)` | Anthropic | GA; `parsed_output`; check `stop_reason` first |
| `responses.parse(text_format=Model)` | OpenAI | `output_parsed` |
| `response_schema=Model` | Gemini | `response.parsed` |
| Instructor `from_provider` | Many | Library-side retries; check model compatibility for TOOLS mode |
| `ai_extract` (v2.1) | Databricks | Optional citations and confidence scores; ≤ 256 fields, ≤ 12 nesting levels |
| Forced tool call | Anthropic | 400 on Opus 5.5 / Sonnet 5.5 / Fable 5.1 — do not use for extraction |

## Common Mistakes

### Wrong

```python
# Vague prompt, no schema, model guesses missing values
prompt = "Extract data from this invoice and return JSON."
```

### Correct

```python
# Schema with descriptions + explicit null policy + API-enforced output + business-rule validation
invoice = extract_invoice(document_text)
```

## Extraction Pipeline

```text
Document ──► (PDF? → native document input or ai_parse_document) ──► structured-output call
         ──► Pydantic business validation ──► OK → store │ fail → re-ask with the validation error, or flag for review
```

## Related

- [Output Formatting](../concepts/output-formatting.md)
- [Document Extraction Pattern](../patterns/document-extraction.md)
- [Multi-Pass Extraction](../patterns/multi-pass-extraction.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.openai.com/docs/guides/structured-outputs
- https://python.useinstructor.com/integrations/anthropic/
- https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_extract

# Document Extraction Pattern

> **Purpose**: Production pattern for extracting structured data from documents (invoices, contracts, reports) with schema-enforced output — Claude native PDF input, OpenAI, and Databricks SQL batch
> **MCP Validated**: 2026-10-17

## When to Use

- Extracting typed fields from PDFs, scans or text documents
- Processing invoices, receipts, contracts or forms at low or high volume
- Building automated document pipelines with validation and human review for low-confidence results

## Implementation: Claude (native PDF + structured outputs)

```python
import base64
from typing import Literal, Optional

import anthropic
from pydantic import BaseModel, Field

client = anthropic.Anthropic()

class LineItem(BaseModel):
    description: str
    quantity: Optional[float] = None
    unit_price: Optional[float] = None
    total: Optional[float] = None

class DocumentExtraction(BaseModel):
    document_type: Literal["invoice", "receipt", "contract", "form", "other"]
    document_number: Optional[str] = None
    date: Optional[str] = Field(None, description="YYYY-MM-DD")
    issuer_name: Optional[str] = None
    recipient_name: Optional[str] = None
    total_amount: Optional[float] = Field(None, description="Number only, no currency symbol")
    currency: Optional[str] = Field(None, description="ISO 4217 code")
    line_items: list[LineItem] = Field(default_factory=list)
    notes: Optional[str] = Field(None, description="Ambiguities or anomalies a reviewer should check")

SYSTEM = """You extract data from business documents for an automated ledger.
Use only values printed in the document and leave fields null when absent, because downstream
reconciliation treats guessed values as errors. Record anything ambiguous in notes."""

def extract_pdf(pdf_bytes: bytes) -> DocumentExtraction:
    resp = client.messages.parse(
        model="claude-sonnet-5-5",
        max_tokens=16000,
        system=SYSTEM,
        messages=[{"role": "user", "content": [
            {"type": "document",                      # PDF before the text instruction
             "source": {"type": "base64", "media_type": "application/pdf",
                        "data": base64.standard_b64encode(pdf_bytes).decode()}},
            {"type": "text", "text": "Extract the document fields."},
        ]}],
        output_format=DocumentExtraction,
    )
    if resp.stop_reason != "end_turn":                # refusal or max_tokens: no valid object
        raise RuntimeError(f"extraction stopped: {resp.stop_reason}")
    return resp.parsed_output

def extract_text(document_text: str) -> DocumentExtraction:
    resp = client.messages.parse(
        model="claude-sonnet-5-5", max_tokens=16000, system=SYSTEM,
        messages=[{"role": "user", "content": f"<document>\n{document_text}\n</document>"}],
        output_format=DocumentExtraction,
    )
    return resp.parsed_output
```

Limits for base64 PDFs: 32 MB per request and up to 600 pages (100 on 200K-context models). For the same file across
many requests, upload once with the Files API (`client.files.upload`) and reference it by `file_id`.

## Implementation: OpenAI (Responses API)

```python
from openai import OpenAI

def extract_text_openai(document_text: str) -> DocumentExtraction:
    resp = OpenAI().responses.parse(
        model="gpt-6.1-sol",
        input=[{"role": "system", "content": SYSTEM},
               {"role": "user", "content": document_text}],
        text_format=DocumentExtraction,
    )
    return resp.output_parsed
```

## Implementation: Databricks SQL batch (`ai_parse_document` + `ai_extract`)

```sql
-- ai_parse_document: DBR 17.3+ (serverless env v3+); ai_extract: DBR 15.4 LTS+, v2.1 schema output.
-- Run once per file and persist: every call is billed inference.
CREATE OR REPLACE TABLE main.docs.invoice_fields AS
WITH parsed AS (
  SELECT path, ai_parse_document(content, map('version', '2.0')) AS doc
  FROM read_files('/Volumes/main/docs/landing/', format => 'binaryFile')
),
text AS (
  SELECT path,
         concat_ws('\n', transform(variant_get(doc, '$.document.elements', 'ARRAY<VARIANT>'),
                                   e -> variant_get(e, '$.content', 'STRING'))) AS body,
         doc:error_status AS parse_errors              -- array of per-page errors; keep for review
  FROM parsed
)
SELECT path, parse_errors,
       r:response:document_number:value::STRING AS document_number,
       r:response:total_amount:value::DOUBLE    AS total_amount,
       r:response:currency:value::STRING        AS currency,
       r:error_message::STRING                  AS extract_error
FROM (
  SELECT path, parse_errors, ai_extract(body,
    '{"document_number": {"type": "string"}, "total_amount": {"type": "number"},
      "currency": {"type": "string", "description": "ISO 4217 code"}}',
    options => map('version', '2.1')) AS r
  FROM text WHERE body IS NOT NULL AND body <> ''
);
```

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `model` | `claude-sonnet-5-5` | `claude-haiku-5-5` for simple, high-volume forms; `claude-opus-5-5` for hard layouts |
| `max_tokens` | `16000` | Room for thinking plus the JSON; current models think by default |
| Output contract | `output_format=DocumentExtraction` | API-enforced; no schema text in the prompt, no JSON mode |
| Retries | 1 on validation error | Re-ask with the Pydantic error message; then route to review |
| Batch | Message Batches API | 50% cheaper for non-interactive backfills |

## Example Usage

```python
sample_invoice = """INVOICE #INV-2024-0847   Date: January 15, 2024
From: Acme Corp   To: Widget Inc
1. Cloud Hosting (Monthly) - 3 units @ $299.00 = $897.00
2. SSL Certificate - 1 unit @ $49.99 = $49.99
Total: $946.99 USD"""

result = extract_text(sample_invoice)
print(result.model_dump_json(indent=2))
# {"document_type": "invoice", "document_number": "INV-2024-0847", "date": "2024-01-15",
#  "issuer_name": "Acme Corp", "total_amount": 946.99, "currency": "USD", ...}
```

## See Also

- [Structured Extraction Concept](../concepts/structured-extraction.md)
- [Multi-Pass Extraction](../patterns/multi-pass-extraction.md)
- [Validation Prompts](../patterns/validation-prompts.md)

## Sources

- https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- https://platform.claude.com/docs/en/build-with-claude/pdf-support
- https://platform.claude.com/docs/en/build-with-claude/files
- https://platform.openai.com/docs/guides/structured-outputs
- https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_parse_document
- https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_extract

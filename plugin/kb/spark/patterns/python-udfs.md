# Python UDFs

> **Purpose**: Choose the right kind of Python function in PySpark 4.x — built-ins first, then Arrow-optimized `udf`, `pandas_udf`, and `arrow_udf` (4.1+) — with version availability for DBR 17.3 LTS / 18 LTS / 19
> **Confidence**: 0.90
> **MCP Validated**: 2026-10-17

## When to Use

- No built-in function or SQL expression can express the logic (check `pyspark.sql.functions` first)
- You need a Python library (parsing, hashing, ML scoring) inside a DataFrame transformation
- Replacing slow row-at-a-time `@udf` code

## Implementation

```python
import pandas as pd
import pyarrow as pa
import pyarrow.compute as pc
from pyspark.sql import functions as F
from pyspark.sql.functions import pandas_udf, udf

# 0. Prefer built-ins: JVM/Photon execution, Catalyst-visible, no Python serialization
df = df.withColumn("email_clean", F.lower(F.trim("email")))

# 1. Arrow-optimized scalar Python UDF (Spark 3.5+; Arrow becomes the default in Spark 4.2)
#    The function still receives ONE Python value per row; Arrow only speeds up (de)serialization.
@udf(returnType="string", useArrow=True)
def mask_email(email):
    if email is None:
        return None
    user, _, domain = email.partition("@")
    return user[:1] + "***@" + domain

# 2. pandas UDF (vectorized, pandas.Series in/out) — fully supported, not legacy
@pandas_udf("double")
def to_usd(amount: pd.Series, rate: pd.Series) -> pd.Series:
    return amount * rate

# 3. Arrow UDF (Spark 4.1+; NOT available on DBR 17.3 LTS / Spark 4.0): pyarrow.Array in/out
from pyspark.sql.functions import arrow_udf   # ImportError on Spark 4.0

@arrow_udf("string")
def lower_arrow(s: pa.Array) -> pa.Array:
    return pc.utf8_lower(s)

out = (df
    .withColumn("email_masked", mask_email("email"))
    .withColumn("amount_usd", to_usd("amount", "exchange_rate"))
    .withColumn("email_lower", lower_arrow("email")))
```

## Configuration

| Kind | Input per call | Min version | DBR 17.3 LTS | DBR 18 LTS | DBR 19 |
|------|----------------|-------------|--------------|------------|--------|
| Built-in function / SQL expression | — | — | Yes | Yes | Yes |
| SQL UDF (`CREATE FUNCTION … RETURN expr`) | — | Spark 4.0 | Yes | Yes | Yes |
| `@udf` (pickled) | one row | 1.x | Yes | Yes | Arrow by default |
| `@udf(useArrow=True)` | one row, Arrow transport | 3.5 | Yes | Yes | Yes (default) |
| `@pandas_udf` | `pandas.Series` batch | 3.0 (type-hint style) | Yes | Yes | Yes |
| `@arrow_udf` | `pyarrow.Array` batch | 4.1 | **No** | Yes | Yes |
| `@udf` with `pa.Array` type hints (vectorized) | `pyarrow.Array` batch | 4.1 | **No** | Yes | Yes |

| Platform rule | Detail |
|---------------|--------|
| Photon | UDFs are not supported by Photon — the stage falls back to the JVM engine |
| Spark Connect (standard/serverless) | UDF serialization happens at execution; outer variables are read at run time — bind them with a factory function |
| Serverless | UDFs cannot access the internet; user code (UDFs, `mapPartitions`) is limited to 1 GB of memory |
| Batch length | Vectorized UDFs receive internal batches, not the whole column — never assume global order or size |

## Example Usage

```python
# Version-safe import for code that must run on DBR 17.3 LTS and newer runtimes
try:
    from pyspark.sql.functions import arrow_udf   # Spark 4.1+
except ImportError:
    arrow_udf = None

def _lower_pd(s: pd.Series) -> pd.Series:      # type hints select the UDF flavour
    return s.str.lower()

lower_fn = pandas_udf(_lower_pd, "string")

if arrow_udf is not None:
    def _lower_pa(s: pa.Array) -> pa.Array:
        return pc.utf8_lower(s)
    lower_fn = arrow_udf(_lower_pa, "string")
```

### Wrong

```python
# useArrow=True does NOT hand you Arrow arrays: `email` is a Python str here
@udf(returnType="string", useArrow=True)
def normalize_email(email):
    return pc.utf8_lower(email)   # pyarrow compute on a scalar str — wrong tool
```

## See Also

- [dataframe-api](../concepts/dataframe-api.md)
- [photon](../concepts/photon.md)
- [spark-connect](../concepts/spark-connect.md)
- [version-matrix](../reference/version-matrix.md)

## Sources

- https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.arrow_udf.html
- https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.functions.udf.html
- https://spark.apache.org/releases/spark-release-4.1.0.html
- https://spark.apache.org/releases/spark-release-4-2-0.html
- https://docs.databricks.com/aws/en/compute/photon
- https://docs.databricks.com/aws/en/spark/connect-vs-classic
- https://docs.databricks.com/aws/en/compute/serverless/limitations

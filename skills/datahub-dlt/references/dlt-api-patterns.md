# dlt API Patterns — DataHub Curated Reference

> **PLACEHOLDER:** This file should be replaced with real content curated from
> the upstream dlthub skill and filtered through DataHub conventions.
> Current content is illustrative only.

## Source and Resource Decorators

### @dlt.source

Defines a logical grouping of related resources (API endpoints).
One source per external system (e.g., ServiceNow, Twilio, NICE).

```python
import dlt

@dlt.source(name="servicenow")
def servicenow_source(
    instance_url: str = dlt.config.value,
    api_key: str = dlt.secrets.value,
):
    """ServiceNow API source — loads incidents and change requests."""
    yield incidents(instance_url, api_key)
    yield change_requests(instance_url, api_key)
```

**DataHub convention:** Source name must match the directory name.
`p8e-data-source-servicenow` → `@dlt.source(name="servicenow")`.

### @dlt.resource

Defines a single data stream (typically one API endpoint = one table).

```python
@dlt.resource(
    name="incidents",
    write_disposition="merge",
    primary_key="sys_id",
)
def incidents(
    instance_url: str,
    api_key: str,
    updated_at: dlt.sources.incremental[str] = dlt.sources.incremental(
        cursor_path="sys_updated_on",
        initial_value="2024-01-01T00:00:00Z",
    ),
):
    """Load ServiceNow incidents with incremental updates."""
    client = create_rest_client(instance_url, api_key)
    yield from client.paginate(
        "/api/now/table/incident",
        params={"sysparm_query": f"sys_updated_on>{updated_at.last_value}"},
    )
```

## REST API Client

Use `dlt.sources.rest_api` for all REST API integrations.

```python
from dlt.sources.rest_api import RESTAPIConfig, rest_api_source

config: RESTAPIConfig = {
    "client": {
        "base_url": "https://instance.service-now.com",
        "auth": {
            "type": "api_key",
            "name": "Authorization",
            "api_key": api_key,
            "location": "header",
        },
    },
    "resources": [
        {
            "name": "incidents",
            "endpoint": {
                "path": "/api/now/table/incident",
                "paginator": "offset",
                "params": {
                    "sysparm_limit": 100,
                },
            },
            "primary_key": "sys_id",
            "write_disposition": "merge",
        },
    ],
}
```

## Incremental Loading

### Cursor-based (preferred for most APIs)

```python
@dlt.resource
def items(
    updated_at: dlt.sources.incremental[str] = dlt.sources.incremental(
        cursor_path="updated_at",
        initial_value="2024-01-01T00:00:00Z",
    ),
):
    # dlt automatically tracks the last cursor value between runs
    yield from fetch_items(since=updated_at.last_value)
```

### Full refresh (when incremental isn't possible)

```python
@dlt.resource(write_disposition="replace")
def static_lookup_table():
    yield from fetch_all_items()
```

**DataHub convention:** Always prefer incremental loading. Use full refresh
only for small lookup/reference tables that don't have reliable timestamps.

## Write Dispositions

| Disposition | Use When | Effect |
|-------------|----------|--------|
| `merge` | Data has a primary key and supports updates | Upserts rows by primary key |
| `append` | Event/log data that only grows | Adds new rows, never modifies existing |
| `replace` | Small lookup tables, full refresh needed | Drops and recreates table each run |

**DataHub convention:** Default to `merge` for entity data, `append` for
event/log data. Document the choice in the source's docstring.

## Configuration and Secrets

dlt resolves configuration in this order:
1. Environment variables (DataHub standard — backed by Azure Key Vault)
2. `secrets.toml` / `config.toml` (local development only)
3. Function parameter defaults

**DataHub convention:** Production secrets ALWAYS come from environment
variables backed by Key Vault. Never commit secrets.toml. Add it to
.gitignore if it exists for local dev.

```python
# dlt automatically resolves these from env vars:
# SOURCES__SERVICENOW__INSTANCE_URL
# SOURCES__SERVICENOW__API_KEY

@dlt.source
def servicenow_source(
    instance_url: str = dlt.config.value,  # resolved from env
    api_key: str = dlt.secrets.value,       # resolved from env (secret)
):
    ...
```

## Schema Inference

dlt infers schemas from the first batch of data. For Databricks destinations:

- JSON objects → STRUCT columns
- Arrays → ARRAY columns
- Nested objects are flattened with `__` separator by default
- Column names are normalized to snake_case

**DataHub convention:** Let dlt handle schema inference. Do NOT manually
define schemas unless there's a specific mismatch issue. If schema issues
arise, use `@dlt.resource(columns=...)` to override specific columns.

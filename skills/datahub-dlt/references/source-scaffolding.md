# dlt Source Scaffolding — DataHub Conventions

> **PLACEHOLDER:** Replace example code with actual patterns from
> p8e-data-source-nice and p8e-data-source-twilio once available.

## Directory Structure

Every dlt source follows this exact layout:

```
sources/p8e-data-source-{name}/
├── pyproject.toml
├── src/
│   └── p8e_data/
│       └── sources/
│           └── {name}/
│               ├── __init__.py          # Source and resource definitions
│               └── config.py            # API config, endpoints, auth helpers
└── tests/
    └── test_{name}_source.py            # Tests
```

**Naming rules:**
- Directory: `p8e-data-source-{name}` (kebab-case)
- Python package: `p8e_data.sources.{name}` (snake_case)
- dlt source name: `{name}` (snake_case, matches package)

## pyproject.toml

```toml
[project]
name = "p8e-data-source-{name}"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "dlt[az,databricks]>=1.14.0,<2.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/p8e_data"]
```

After creating the source, add it to the root `pyproject.toml`:

```toml
[tool.uv.workspace]
members = [
    "sources/p8e-data-source-nice",
    "sources/p8e-data-source-twilio",
    "sources/p8e-data-source-{name}",    # ← add this
]
```

## __init__.py Template

```python
"""
dlt source for {Name} API.

Loads {describe what entities} into the DataHub Raw datalake layer.
Data lands in: {catalog}.raw_{name}.{table_names}

Auth: {OAuth2 | API key | etc.} via Azure Key Vault environment variables.
Loading: Incremental via {cursor field} / Full refresh for {lookup tables}.
"""

import dlt
from dlt.sources.rest_api import RESTAPIConfig, rest_api_source


@dlt.source(name="{name}")
def {name}_source(
    base_url: str = dlt.config.value,
    api_key: str = dlt.secrets.value,
):
    """
    {Name} API source.

    Yields:
        {entity_1}: {description}
        {entity_2}: {description}
    """
    yield {entity_1}(base_url, api_key)
    yield {entity_2}(base_url, api_key)


@dlt.resource(
    name="{entity_1}",
    write_disposition="merge",
    primary_key="id",
)
def {entity_1}(
    base_url: str,
    api_key: str,
    updated_at: dlt.sources.incremental[str] = dlt.sources.incremental(
        cursor_path="updated_at",
        initial_value="2024-01-01T00:00:00Z",
    ),
):
    """{Entity_1} from {Name} API with incremental loading."""
    # Implementation here
    ...
```

## config.py Template

```python
"""
{Name} API configuration and auth helpers.
"""

# API endpoints
ENDPOINTS = {
    "{entity_1}": "/api/v1/{entity_1}",
    "{entity_2}": "/api/v1/{entity_2}",
}

# Pagination defaults
DEFAULT_PAGE_SIZE = 100
MAX_RETRIES = 3

# Rate limiting
RATE_LIMIT_REQUESTS = 100
RATE_LIMIT_WINDOW_SECONDS = 60
```

## Test Template

```python
"""Tests for p8e-data-source-{name}."""

import pytest


class TestSourceDiscovery:
    """Verify the source can be imported and discovered."""

    def test_source_import(self):
        from p8e_data.sources.{name} import {name}_source
        assert callable({name}_source)

    def test_source_returns_resources(self):
        from p8e_data.sources.{name} import {name}_source
        # Source should be callable and return resources
        # Note: this may require mock credentials
        source = {name}_source(
            base_url="https://mock.example.com",
            api_key="test-key",
        )
        resource_names = [r.name for r in source.resources.values()]
        assert "{entity_1}" in resource_names
        assert "{entity_2}" in resource_names


class TestResourceSchemas:
    """Verify resource schemas are inferred correctly."""

    def test_{entity_1}_schema(self):
        # Test with mock data that schema inference produces expected columns
        ...

    def test_{entity_2}_schema(self):
        ...
```

## Verification Checklist

After scaffolding, run:

```bash
# From the source directory
cd sources/p8e-data-source-{name}

# Tests pass
uv run pytest

# Linting clean
uv run ruff check .
uv run ruff format --check .
```

Then verify from the workspace root:

```bash
# Workspace resolves the new member
uv sync

# Source is importable
uv run python -c "from p8e_data.sources.{name} import {name}_source; print('OK')"
```

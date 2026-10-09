# Running Tests

## Testing Strategy

- **Unit tests** (`tests/unit/`) - Fast, mocked dependencies, run during development
- **Integration tests** (`tests/integration/`) - Slower end-to-end tests. Some of them run against an embedded Milvus Lite database
  - Includes **acceptance tests** - end-to-end tests via MCP protocol using production data
- **Coverage requirement** - 100% enforced by `./pytest.sh --check-coverage`

## Quick Reference

```bash
# Run all tests (unit + integration)
./pytest.sh

# Run with 100% coverage check
./pytest.sh --check-coverage

# Run only unit tests (fast iteration)
./pytest.sh tests/unit/

# Run only integration tests
./pytest.sh tests/integration/

# Run specific test file
./pytest.sh tests/integration/test_mcp_tools_acceptance.py
```

## Why pytest.sh?

The `./pytest.sh` script runs ALL tests by default. It also:
- Auto-activates `.venv`
- Sets PYTHONPATH correctly
- Provides `--check-coverage` flag for CI/CD

Use `./pytest.sh tests/unit/` for quick unit-test-only iterations during development.

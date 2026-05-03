# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**shcheck** is a security header checker tool that analyzes HTTP response headers from target websites to identify which security headers are present, missing, or improperly configured. It supports both single-target and bulk scanning (via host file), with colored terminal output and JSON export options.

## Architecture

### Code Structure
```
shcheck/
├── __init__.py          # Package init (minimal)
├── shcheck.py           # Main monolithic module (~480 lines)
└── tests/
    ├── __init__.py
    └── test_shcheck.py  # Comprehensive test suite (~420 lines)
```

### Key Design Patterns

**Single-module architecture**: All functionality lives in `shcheck/shcheck.py`. There are no submodules — this is intentional for simplicity and standalone script portability.

**Global state**: The `options` namespace (from `argparse`) is globally accessible across functions. Tests set `shcheck.options` directly via `make_options()` helper. When adding new options, update both the `parse_options()` function and the test helpers.

**Header categorization**: Headers are defined in module-level dictionaries:
- `sec_headers` — Security headers with severity levels (`error`, `warning`, `deprecated`)
- `information_headers` — Information disclosure headers (Server, X-Powered-By, etc.)
- `cache_headers` — Caching-related headers
- `client_headers` — Default request headers sent to targets

**JSON vs text output**: Dual output mode controlled by `options.json_output`. The `log()` function suppresses output when JSON mode is active to avoid corrupting JSON on stdout.

## Development Workflow

### Running Tests

```bash
# Install test dependencies
pip install pytest
pip install -e .

# Run all tests
pytest tests/ -v

# Run a specific test
pytest tests/test_shcheck.py::test_check_target_success -v

# Run tests across all Python versions (CI matrix)
# The CI tests 3.9, 3.10, 3.11, 3.12, 3.13
```

### Running the Tool

```bash
# From source
./shcheck.py https://example.com

# Installed via pip
shcheck.py https://example.com

# With options
./shcheck.py -j -i -x -k https://example.com

# Bulk scan from file
./shcheck.py --hfile hosts.txt
```

### Adding New Tests

Follow the existing patterns in `tests/test_shcheck.py`:
- Use `make_options()` for consistent option setup
- Use `_mock_response()` for creating mock HTTP responses
- Use `_run_json()` / `_run_normal()` for integration-style tests
- Tests mock `urllib.request.urlopen` to avoid network calls

**Important**: Tests rely on `shcheck.options` being set. Always set it before testing functions that reference it (like `colorize()`).

### Building and Distribution

```bash
# Install in development mode
pip install -e .

# Build for PyPI (requires build tools)
python setup.py sdist bdist_wheel
```

The `setup.py` pulls the long description from `README.md` and defines version `1.7`.

## Key Implementation Details

### Header Processing

Headers are normalized to lowercase keys via `parse_headers()` to handle case-insensitive HTTP header names. When checking for specific headers, always compare lowercase.

### X-Frame-Options and CSP Interaction

The tool intelligently handles the interaction between `X-Frame-Options` and CSP's `frame-ancestors` directive. If CSP with `frame-ancestors` is present, `X-Frame-Options` is excluded from the check (per modern web standards). **Critical**: The code uses `dict(sec_headers)` to create a copy per-target, preventing mutation across multiple targets in a single run.

### Color System

- `--colours=dark` (default): Yellow warnings (`\033[93m`)
- `--colours=light`: Magenta warnings (`\033[95m`)
- `--colours=none`: No ANSI codes

The `colorize()` function depends on the global `options` object. Always set `shcheck.options` when testing colorize.

### JSON Output Format

```json
{
  "https://example.com": {
    "present": { "Header-Name": "value" },
    "missing": ["Missing-Header"],
    "information_disclosure": { ... },
    "caching": { ... }
  }
}
```

`information_disclosure` and `caching` are only included when `-i` or `-x` flags are used.

### Multi-target Processing

When scanning multiple targets (via host file or command-line list), each target is processed sequentially with a fresh opener per target. The `sec_headers` dictionary is copied per-target to prevent cross-contamination.

## Common Modification Patterns

### Adding a New Security Header

1. Add to `sec_headers` dictionary in `shcheck/shcheck.py`:
   ```python
   'New-Header-Name': 'warning',  # or 'error', 'deprecated'
   ```

2. Add corresponding test cases in `tests/test_shcheck.py`

### Adding a New CLI Option

1. Add to `parse_options()` using `parser.add_argument()`
2. Update the `main()` function to handle the new option
3. Add test coverage

### Changing Header Severity

Modify the value in the `sec_headers` dictionary. Severity affects:
- Color coding in terminal output
- Default visibility for deprecated headers (hidden unless `-k` is used)

## Testing Focus Areas

The test suite emphasizes:
- **Regression tests**: Several tests are marked as failing and await fixes
- **Per-target isolation**: Ensuring headers from one target don't affect others
- **JSON/text consistency**: Verifying both output modes agree
- **Edge cases**: HTTP vs HTTPS, deprecated headers, information disclosure

See test functions starting with `test_bug` and `test_upgrade_*` for known issues.

## CI/CD

GitHub Actions runs tests on every push/PR across Python 3.9–3.13 (see `.github/workflows/ci.yml`). The workflow:
1. Checks out code
2. Sets up Python
3. Installs `pytest` and the package in editable mode
4. Runs `pytest tests/ -v`

## Dependencies

- **Standard library only**: `urllib`, `http.client`, `ssl`, `argparse`, `json`, `socket`, `sys`
- **Test dependencies**: `pytest` (via `requirements.txt` or CI install)

No external runtime dependencies — this is a design feature for easy deployment.

## Troubleshooting

**Tests fail due to options not being set**: Ensure `shcheck.options` is initialized via `make_options()` before calling any function that references it.

**JSON output is corrupted**: Verify that `print()` calls go to stderr or are gated by `if not options.json_output`. The `log()` helper handles this automatically.

**SSL errors in testing**: Tests mock `urlopen`, but if you need to bypass SSL validation in actual use, use the `-d` flag.

## License

GPLv3 — see LICENSE.txt
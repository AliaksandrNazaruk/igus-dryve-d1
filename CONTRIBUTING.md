# Contributing

Thank you for considering a contribution to the igus Dryve D1 service!

## Getting Started

```bash
git clone https://github.com/AliaksandrNazaruk/igus-dryve-d1.git
cd igus-dryve-d1
python -m venv .venv
source .venv/bin/activate   # Linux/macOS
# .venv\Scripts\activate    # Windows
pip install -r requirements-dev.txt
```

## Running Tests

Service-level tests (no hardware required):

```bash
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 python -m pytest -q -p pytest_asyncio.plugin tests -m "not simulator"
```

Driver unit tests:

```bash
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 python -m pytest -q -p pytest_asyncio.plugin drivers/tests/unit -m "not simulator"
```

## Linting & Type Checking

```bash
# App & service
python -m ruff check main.py app tests
python -m mypy main.py app

# Driver
python -m ruff check drivers/dryve_d1
python -m mypy drivers/dryve_d1
```

## Local CI Parity

Run the full quality gate locally (mirrors GitHub Actions):

```bash
bash run_ci_local.sh
```

## Pull Request Process

1. Fork the repository and create a feature branch.
2. Ensure all checks pass (`bash run_ci_local.sh`).
3. Fill in the [pull request template](.github/pull_request_template.md).
4. Keep PRs focused — one logical change per PR.

## Code Style

- Enforced by [ruff](https://docs.astral.sh/ruff/) (config in `pyproject.toml`).
- Type hints are expected on all public APIs.
- Static analysis via [mypy](https://mypy-lang.org/) must pass.

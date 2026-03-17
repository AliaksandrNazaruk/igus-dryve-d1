# igus Dryve D1 Motor Control Service

[![CI](https://github.com/AliaksandrNazaruk/igus-dryve-d1/actions/workflows/ci.yml/badge.svg)](https://github.com/AliaksandrNazaruk/igus-dryve-d1/actions/workflows/ci.yml)
[![Python 3.12+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Code style: ruff](https://img.shields.io/badge/code%20style-ruff-000000.svg)](https://docs.astral.sh/ruff/)

Production-grade **FastAPI** microservice for controlling [igus Dryve D1](https://www.igus.com/info/drive-technology-dryve-d1) stepper/servo motors via **Modbus/TCP**. Implements the **CiA 402** (CANopen Drive Profile) state machine over a pure-Python async Modbus transport.

## Features

- **CiA 402 state machine** — enable, disable, fault reset, homing, profile position, profile velocity, jog
- **REST API (v1)** — versioned endpoints for all motion commands with Pydantic validation
- **Real-time SSE stream** — telemetry and command status events at `/drive/events`
- **Prometheus metrics** — 30+ drive health gauges and operation counters at `/metrics`
- **Health scoring** — configurable weighted algorithm with Kubernetes-ready `/ready` probe
- **API key authentication** — timing-safe HMAC comparison (optional, for production)
- **Legacy API lifecycle** — deprecation → sunset → removed phases with `Sunset` headers
- **Request tracing** — `request_id`, `command_id`, `op_id` across HTTP, SSE, and driver logs
- **Bundled Modbus TCP simulator** — develop and test without hardware
- **Interactive control panel** — static HTML/JS dashboard

## Architecture

```
┌─────────────┐       ┌──────────────────────────┐       ┌──────────────────┐
│   Client    │ HTTP  │     FastAPI Service       │ Modbus│  Dryve D1 Motor  │
│  REST/SSE   │──────>│  main.py + app/           │──TCP──│  (or simulator)  │
└─────────────┘       └────────────┬─────────────┘       └──────────────────┘
                                   │
                      ┌────────────┴─────────────┐
                      │    DryveD1 Driver         │
                      │    drivers/dryve_d1/      │
                      │                           │
                      │  ┌─────────────────────┐  │
                      │  │ CiA 402 State Machine│  │
                      │  └─────────────────────┘  │
                      │  ┌─────────────────────┐  │
                      │  │ Modbus/TCP Transport │  │
                      │  └─────────────────────┘  │
                      │  ┌─────────────────────┐  │
                      │  │ Telemetry Poller     │  │
                      │  └─────────────────────┘  │
                      └───────────────────────────┘
```

## Quick Start

### Docker (recommended)

```bash
cp .env.example .env
# Edit .env — set IGUS_MOTOR_IP to your drive's address
docker compose up -d --build
```

Open Swagger UI: http://localhost:8101/docs

### Local development

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
python main.py
```

### Using the bundled simulator

```bash
python simulator.py &                  # starts Modbus TCP simulator on port 501
export IGUS_MOTOR_IP=127.0.0.1
export IGUS_MOTOR_PORT=501
export DRYVE_UNIT_ID=0
python main.py
```

> The simulator responds with Unit ID 0 in MBAP headers. Set `DRYVE_UNIT_ID=0` or enable `DRYVE_ALLOW_UNIT_ID_WILDCARD=1`.

## API Endpoints

### Motion control (API v1)

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/drive/move_to_position` | Profile position move |
| `POST` | `/drive/jog` | Jog — hold-to-move with velocity |
| `POST` | `/drive/reference` | Homing sequence |
| `POST` | `/drive/stop` | Controlled deceleration |
| `POST` | `/drive/quick_stop` | Emergency stop |
| `POST` | `/drive/fault_reset` | Clear drive fault state |
| `GET` | `/drive/status` | Current drive telemetry |
| `GET` | `/drive/events` | SSE telemetry stream |
| `GET` | `/drive/trace/latest` | Last command trace |

### System

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/ready` | Readiness probe (503 when degraded) |
| `GET` | `/metrics` | Prometheus-format metrics |
| `GET` | `/info` | Service version and build info |
| `GET` | `/docs` | Swagger UI |

### Legacy endpoints (deprecated)

`/move`, `/reference`, `/fault_reset`, `/status`, `/position`, `/is_motion` — emit `Deprecation` and `Sunset` headers. Migration target: `/drive/*`.

## Configuration

Copy [`.env.example`](.env.example) and adjust for your setup. Key variable groups:

| Group | Variables | Description |
|-------|-----------|-------------|
| **Drive endpoint** | `IGUS_MOTOR_IP`, `IGUS_MOTOR_PORT`, `DRYVE_UNIT_ID` | Modbus/TCP target |
| **Runtime profile** | `DRYVE_PROFILE` | `production` (strict) or `simulator` (tolerant) |
| **Authentication** | `IGUS_API_KEY`, `IGUS_AUTH_DISABLED` | X-API-Key header |
| **Health tuning** | `DRYVE_HEALTH_WEIGHT_*` | Health score penalty weights |
| **Legacy lifecycle** | `LEGACY_API_PHASE`, `LEGACY_API_SUNSET` | `deprecated` → `sunset` → `removed` |
| **Observability** | `LOG_LEVEL`, `DRYVE_STATUS_EVENT_THROTTLE_S` | Logging and SSE throttle |

## Observability

### Prometheus metrics

Key metrics exposed at `GET /metrics`:

- `igus_drive_connected` — connection state (0/1)
- `igus_drive_health_score` — aggregate health (0–100)
- `igus_drive_fault_active` — drive fault bit
- `igus_drive_telemetry_stale` — telemetry freshness flag
- `igus_drive_operation_errors_total{operation,code,status}` — error counters
- `igus_legacy_api_requests_total{path,phase}` — legacy endpoint usage

See [`monitoring/`](monitoring/) for Prometheus alert rules and incident runbook.

### Recommended alerts

- `igus_drive_connected == 0` for > 10 s
- `igus_drive_telemetry_stale == 1` for > 10 s
- `increase(igus_drive_operation_errors_total[5m]) > 0`

### Request tracing

API responses include `request_id` and `command_id`. SSE `type=command` events carry `op_id` for end-to-end correlation through HTTP → SSE → driver logs.

## Project Structure

```
.
├── main.py                  # FastAPI entry point
├── app/                     # Service layer
│   ├── api_routes.py        #   API v1 endpoints
│   ├── routes.py            #   Legacy endpoints
│   ├── system_routes.py     #   /ready, /metrics, /info
│   ├── application/         #   Use cases, commands, DTOs
│   ├── domain/              #   Health scoring
│   └── static/              #   Control panel HTML
├── drivers/
│   └── dryve_d1/            # Standalone Modbus/CiA 402 driver
│       ├── api/             #   High-level async facade
│       ├── cia402/          #   State machine implementation
│       ├── motion/          #   Motion profiles (position, velocity, jog, homing)
│       ├── protocol/        #   Modbus/CANopen telegram codec
│       ├── transport/       #   TCP client, session, retry
│       └── telemetry/       #   Status polling & snapshots
├── tests/                   # Service-level tests (24 files)
├── drivers/tests/           # Driver tests (unit, integration, property-based)
├── simulator.py             # Modbus TCP simulator
├── monitoring/              # Prometheus alert rules & runbook
├── Dockerfile
├── docker-compose.yml
└── .github/workflows/ci.yml # CI pipeline
```

## Development & Testing

```bash
# Install dev dependencies
pip install -r requirements-dev.txt

# Run service tests
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 python -m pytest -q -p pytest_asyncio.plugin tests -m "not simulator"

# Run driver unit tests
PYTEST_DISABLE_PLUGIN_AUTOLOAD=1 python -m pytest -q -p pytest_asyncio.plugin drivers/tests/unit -m "not simulator"

# Lint & type check
python -m ruff check main.py app tests drivers/dryve_d1
python -m mypy main.py app
python -m mypy drivers/dryve_d1

# Full local CI parity
bash run_ci_local.sh
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full development guide.

## License

[MIT](LICENSE) &copy; 2026 Aliaksandr Nazaruk

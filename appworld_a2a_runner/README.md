# AppWorld A2A Proxy Runner

A standalone Python runner that uses AppWorld to enumerate tasks and fetch task text, then calls a remote agent over A2A (Agent-to-Agent) protocol with a plain-text prompt, collects the plain-text response, and emits OpenTelemetry (OTEL) telemetry.

## Features

- **Standalone execution**: Not integrated with AppWorld's experiment framework
- **Sequential task processing**: One task at a time for simplicity
- **A2A protocol support**: Communicates with remote agents using the A2A protocol via JSON-RPC over HTTP
- **OpenTelemetry instrumentation**: Comprehensive traces, metrics, and logs
- **Strict failure handling**: Any error or timeout marks the task as failed
- **Configurable via environment variables**: Easy deployment and configuration
- **No SDK dependencies**: Uses plain HTTP requests instead of the A2A SDK to avoid dependency conflicts with AppWorld

## Installation

### Prerequisites

- Python 3.11 or higher
- Access to an A2A-compatible agent endpoint

### Install from source

```bash
git clone <repository-url>
cd appworld_a2a_runner
uv venv
source .venv/bin/activate
uv sync
```

## Configuration

```
cp example.env .env
```
Configure the .env file as needed

| Environment Variable | Default Setting | Required? |
| --- | --- | --- |
| `A2A_BASE_URL` | `(none)` | Yes |
| `A2A_TIMEOUT_SECONDS` | `300` | No |
| `A2A_AUTH_TOKEN` | `(none)` | No |
| `A2A_VERIFY_TLS` | `true` | No |
| `A2A_ENDPOINT_PATH` | `/v1/chat` | No |
| `APPWORLD_DATASET` | `(none)` | Yes |
| `APPWORLD_REMOTE_APIS_URL` | `(none)` | Yes |
| `APPWORLD_ROOT` | `(none)` | No |
| `MAX_TASKS` | `(none)` | No |
| `ABORT_ON_FAILURE` | `false` | No |
| `OTEL_SERVICE_NAME` | `appworld-a2a-proxy` | No |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `(none)` | No |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` | No |
| `OTEL_RESOURCE_ATTRIBUTES` | `(none)` | No |
| `OTEL_INSTRUMENT_REQUESTS` | `true` | No |
| `LOG_PROMPT` | `0` | No |
| `LOG_RESPONSE` | `0` | No |


## Usage

### Basic Usage

```bash
uv run appworld-a2a-runner
```

## A2A Protocol Format

The runner communicates with A2A endpoints using JSON-RPC over HTTP (no SDK dependencies to avoid conflicts with AppWorld).

### Discovery and Endpoint Resolution

The runner first fetches:

```text
GET {A2A_BASE_URL}/.well-known/agent-card.json
```

Then resolves the RPC URL using this precedence:

1. If agent card `url` has a non-root path, use it directly.
2. Otherwise use `{service_base}{A2A_ENDPOINT_PATH}`.
3. If card fetch fails, use `{A2A_BASE_URL}{A2A_ENDPOINT_PATH}`.

## Output

### Console Summary

At the end of each run, a summary is printed:

```
============================================================
RUN SUMMARY
============================================================
Dataset:           test_normal
Tasks Attempted:   100
Tasks Succeeded:   95
Tasks Failed:      5
Total Wall Time:   1234.56s
Average Latency:   12345.67ms
P50 Latency:       10000.00ms
P95 Latency:       20000.00ms
============================================================
```

### OpenTelemetry Data

The runner emits comprehensive telemetry:

#### Traces

Each task creates a span (`a2a_proxy.task`) with:

**Attributes:**
- `appworld.task_id`: Task identifier
- `appworld.dataset`: Dataset name
- `a2a.base_url`: A2A endpoint URL
- `a2a.timeout_seconds`: Timeout value
- `prompt.chars`: Prompt size in characters
- `response.chars`: Response size in characters
- `task.status`: `success` or `failed`
- `a2a.duration_ms`: End-to-end A2A operation latency in milliseconds

**Child spans:**
- `a2a_proxy.prompt.build`: Prompt construction
- `a2a_proxy.a2a.send_prompt`: End-to-end A2A `send_prompt` call

**Auto-instrumented HTTP spans:**
- Outbound `requests` spans for agent-card discovery, `message/send`, and `tasks/get` calls

**Events:**
- `prompt_built`: When prompt is constructed
- `task_failed`: When task fails (includes error details)

#### Metrics

**Counters:**
- `a2a_proxy_tasks_total{status=success|failed}`: Total tasks processed
- `a2a_proxy_errors_total{error_type=...}`: Total errors by type

**Histograms:**
- `a2a_proxy_task_latency_ms`: End-to-end task latency
- `a2a_proxy_a2a_latency_ms`: A2A request latency
- `a2a_proxy_prompt_size_chars`: Prompt size distribution
- `a2a_proxy_response_size_chars`: Response size distribution

**Gauge:**
- `a2a_proxy_inflight_tasks`: Current tasks in flight (0 or 1)



## Current Limitations

- Sequential execution only (no concurrency)
- No retry mechanism
- No streaming response support
- No structured response parsing

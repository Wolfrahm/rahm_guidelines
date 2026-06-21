# Rahm Logging Guidelines

How Rahm backend services log — the same shape across Python and other languages. Each language gets its own Rahm-built library that implements this contract. Apps emit JSON to stdout; a collector ships it to Loki.

## 1. Principles

- One JSON shape for every service. The library writes; apps describe what happened.
- Loki is the primary target, but the same JSON ships cleanly to other aggregators.
- App developers only pick `severity`, `domain`, `event`, `message`, and any extra context. Transport, formatting, mandatory fields, and timestamps are the library's job.
- PII isn't enforced or redacted today, but might be later — libraries should leave a place for a redaction layer without changing the wire format.

## 2. Configuration

The library reads these environment variables at init.

| Variable             | Sets                                               | Default |
|----------------------|----------------------------------------------------|---------|
| `RAHM_APPLICATION`   | `application` label.                        | `unknown` |
| `RAHM_ENVIRONMENT`   | `environment` label.                        | `unknown` |
| `RAHM_LOG_SEVERITY`  | Minimum severity emitted. Entries below are dropped. Accepts the five severities or `none` (suppresses all output; for tests). | `info`  |
| `RAHM_LOG_FORMAT`    | `json` (wire format), `text` (colored human-readable), or `logfmt` (compact `key=value`, useful for LLM consumption). | `json`  |
| `RAHM_LOG_TRACE_ID`  | `enabled` or `disabled`. When `disabled`, `trace_id` is not bound, propagated, or emitted. | `enabled` |

All env-var *values* are case-insensitive.

## 3. Transport

Apps write JSON lines to **stdout**, one entry per line. A collector (Fluent Bit, the Loki Docker driver, or equivalent) does the shipping. stderr is reserved for crash output.

## 4. Severity

Five levels:

| Severity  | Use when                                                |
|-----------|---------------------------------------------------------|
| `debug`   | Diagnostics. |
| `info`    | Normal events worth recording.                          |
| `warning` | Something unexpected, but the operation still succeeded.|
| `error`   | An operation failed; the process keeps running.         |
| `fatal`   | Unrecoverable; the process is about to exit. |

Severity values are always lower-case strings on the wire. Entries below the minimum severity (`RAHM_LOG_SEVERITY`) are dropped at the source.

## 5. Output formats

The output format is controlled by `RAHM_LOG_FORMAT`. Two rules apply to all modes:

- Timestamps are RFC 3339 UTC with milliseconds: `2026-06-14T08:42:11.123Z`.
- Aim for entries under 64 KiB. If a serialized entry would exceed that, the library replaces the largest top-level field's value with a truncation marker indicating the original byte count, then the next-largest, until the entry fits. Entries are never split or dropped.

### 5.1 json (default)

UTF-8 JSON, one entry per line, ending in `\n`. No pretty-printing, no comments. The canonical wire format.

### 5.2 logfmt

One line of `key=value` pairs per entry, with quoted strings and `\n`-escaped newlines. Useful when an LLM is the primary log consumer.

### 5.3 text

Colored, human-readable output for local development. Trims and re-lays out a few fields:

- `timestamp` is omitted on normal entries.
- `file` and `line` are appended to the message inline — e.g. `user logged in - main.py:85`.
- `severity` may be rendered upper-case for readability.

## 6. Schema

### 6.1 Filter fields

These five fields become Loki stream labels, so they have to stay low-cardinality.

| Field         | Where it's set              | Allowed values                                |
|---------------|-----------------------------|-----------------------------------------------|
| `application` | `RAHM_APPLICATION`          | Short service name, e.g. `rahm_billing_api`.  |
| `environment` | `RAHM_ENVIRONMENT`          | Free-form short string, e.g. `dev`, `staging`, `prod`. |
| `severity`    | Per call                    | One of the five severities.                                       |
| `domain`    | Per call (default `system`) | `system`, `auth`, `metric`, `transaction`.    |
| `event`       | Per call                    | A bounded enum per service, snake_case.       |

`event` names a *general thing that can happen to a resource* — what changed, succeeded, or failed — at a level coarse enough to be a short, enumerable list per service. `message` adds the specifics, in human-readable form. The same `event` can pair with many different `message` values; structured details (IDs, amounts, reasons) go to their own fields, not into `message`. For example:

- `event=customer_create_failed`
  - `message="duplicate email address"`
  - `message="invalid phone format"`
  - `message="country not supported"`

Filter and alert on `event`; read `message` for context. Because `event` is also a filter field, it needs low cardinality per service.

### 6.2 Mandatory fields

Every entry has: `timestamp`, `severity`, `application`, `environment`, `domain` (default `system`), `event`, `message`, `file`, `line`.

`timestamp`, `file`, and `line` are library-set — `file` and `line` are captured from the caller at log time. The severity threshold is checked first; entries below it are dropped before any caller capture, so filtered calls don't pay the stack-walk cost. Library-generated entries for uncaught failures omit `file`/`line`; the stack trace carries the location. `message` is a short single-line human-readable summary; put structured data in other fields. The library replaces any newline in `message` with a space.

### 6.3 Standard fields

Use these names when the concept applies — don't invent synonyms.

| Field         | Meaning                                                                                                                                  |
|---------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `trace_id`    | Correlation ID across services. Set by a scope wrapper or by app code; absent on logs emitted outside any scope (CLI, early init).|
| `resource_id` | Primary resource the entry is about.                                                                                                     |
| `user_id`     | Authenticated user identifier.                                                                                                           |

### 6.4 Custom fields

Apps can add whatever they need, with these conventions:

- snake_case.
- Values: JSON primitives, arrays, or objects.
- Devs can populate standard fields directly.

Recommended prefixes for common domains:

- `http_*` for HTTP request/response details — `http_method`, `http_path`, `http_status`, `http_duration_ms`.
- `db_*` for database calls — `db_statement`, `db_duration_ms`, `db_table`.
- `stream_*` for event-stream context (Kafka, Fluvio, …) — `stream_topic`, `stream_partition`, `stream_offset`.
- `error_*` for exception details on system errors — `error_type`, `error_message`, `error_trace`.
- `job_*` / `run_*` for batch context — `job_id`, `run_id`.

### 6.5 Output customization

A customer's pipeline might expect different conventions — `msg` instead of `message`, a different timestamp format, and so on. Each language library exposes its own idiomatic mechanism for transforming the output just before writing JSON.

- Any field can be renamed or reformatted, standard or custom.
- Default: no renames — canonical names go to the wire.
- All library rules (mandatory fields, collisions, allow-lists, reserved names) operate on canonical names.

## 7. Log calls

Every log call: `rahm.log.<severity>(event, message, **fields)`.

- `event` — required identifier. No spaces; snake_case recommended.
- `message` — required free-form sentence.
- `**fields` — `domain`, standard fields, and custom fields.

Example: `rahm.log.info('user_logged_in', 'user logged in', user_id='u_123')`.

## 8. Log context

Each task (request, message, batch) has at most one scope. The wrapper opens the scope, app code adds or changes fields, and the wrapper closes it at task end. Outside a scope (server startup, CLI tools), log entries carry only mandatory fields plus any per-call args.

### 8.1 The contract

- Inside an active scope, app code can `bind` to add or change fields, and `unbind` to remove them. For a temporary field in a sub-block, use `with rahm.log.bind(field=value): ...` — it auto-unbinds on exit.
- Re-binding a non-mandatory field overwrites the previous value.
- Library-set mandatory fields (`timestamp`, `application`, `environment`, `file`, `line`) can't be bound or passed per call — the library sets them.
- Caller-set mandatory fields (`severity`, `domain`, `event`, `message`) must be passed per call. Binding them to the scope causes an error.
- Per-call args attach to a single log line and aren't stored in the scope. Passing a non-mandatory name that's already in the scope raises an error: `attribute <name> already set in scope`.
- Concurrent units of work must not see each other's context. Libraries use per-task isolation (e.g. `contextvars` in Python, not `threading.local`).

### 8.2 API

- `rahm.log.scope(field=value, ...)` — opens a scope. Must be used as a context manager (`with`). Built-in wrappers (below) use this internally.
- `rahm.log.bind(field=value, ...)` — adds or changes fields in the active scope. Raises an error if no scope is active.
- `rahm.log.unbind('field')` — removes a field from the active scope. Raises an error if no scope is active or the field isn't set.
- `with rahm.log.bind(field=value): ...` — temporary bind; auto-unbinds when the block exits.

The scope is dissolved automatically when the `with rahm.log.scope(...)` block exits — all bound fields are dropped and the task returns to scope-less mode.

Other languages map `with` to their idiom for scoped cleanup.

### 8.3 Built-in wrappers

Each library ships ready-made scope wrappers:

- **HTTP request** (Starlette, Robyn, Falcon, …): binds `trace_id` and HTTP context (e.g. `http_method`, `http_path`) at request entry. `trace_id` comes from the `X-Trace-Id` request header; if absent, the library generates a lowercase ULID. If authentication identifies a user, `user_id` is added.
- **Kafka / Fluvio consumer**: binds `trace_id` (from the `x-trace-id` header, or a generated lowercase ULID if absent) and message context (`stream_*` fields — topic, partition, offset).
- **Batch / job entry**: binds run context (e.g. `job_id`, `run_id`) and a generated `trace_id` (lowercase ULID).

Set `RAHM_LOG_TRACE_ID=disabled` to disable `trace_id` entirely.

App code can add fields to the active scope at any time. For example, bind `resource_id` at the top of a block that operates on a specific resource so every subsequent log line carries it.

### 8.4 Domain-based filtering

Each domain restricts which severities and optional fields appear in an entry by default. At log time the library drops any scope field that isn't in the domain's allow-list. Per-call args bypass the allow-list.

| Domain        | Severities    | Optional fields                                                      |
|---------------|---------------|----------------------------------------------------------------------|
| `system`      | debug–fatal   | All available fields. |
| `auth`        | info, warning | info: `trace_id`, `user_id`. warning: all standard and custom fields.|
| `transaction` | info, warning | `trace_id`, `resource_id`.                                           |
| `metric`      | info, warning | `trace_id`, `resource_id`.                                           |

Rules:

- debug, error, and fatal are only valid with `domain=system`. Otherwise the library coerces `domain` to `system` and emits a warning, so the call site can be fixed.
- Per-call overrides: `include=[fields…]` keeps scope fields the domain would drop; `exclude=[fields…]` drops scope fields the domain would keep. Both names are reserved and can't be used as custom field names.

## 9. Errors and exceptions

When you log a system error or exception, the library captures the relevant details into a set of `error_*` fields. The exact names and count vary by language and runtime — typically the error type/class, message, and stack trace. Stack traces, when present, are escaped strings with newlines as `\n`.

When called inside an `except` block (or equivalent), `rahm.log.error(...)` and `rahm.log.fatal(...)` automatically capture the active exception into `error_*` fields and set `domain=system`. Business failures with expected causes (card declined, validation rejection) are not errors — log them with their natural domain at info or warning.

Example: a payment provider returning 500 or timing out is a system error — `rahm.log.error('payment_api_unreachable', 'payment provider timed out')`. The same provider declining a card is a business failure — `rahm.log.warning('payment_declined', 'payment declined', domain='transaction', reason='insufficient_funds')`.

### 9.1 Uncaught exceptions and panics

The library installs the language's global error handlers on init so any uncaught failure produces a uniform log entry. In Python: `sys.excepthook`, `threading.excepthook`, and the asyncio loop exception handler.

The resulting entry has `domain=system`, `event=uncaught_exception`, with the `error_*` fields set as above. `file` and `line` are omitted — the stack trace carries the location. Severity is `fatal` if the process is about to exit, `error` if it keeps running.

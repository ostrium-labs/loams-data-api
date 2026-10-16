# The Loams Data API, `data-api-1`

**Status: draft v0** (2026-10-16). Nothing here is stable. The version this draft is heading for is `data-api-1.0` (§13).
Origin: Loams design 58 §5 (D1042–D1050). Open points are marked `TODO(Qnnn)` with the design's question number.
Wire contract: [`proto/loams/data/v1/data.proto`](../proto/loams/data/v1/data.proto). Where the proto and this text disagree, this text wins, and the proto has a bug.

## 1. Introduction

The Data API lets an application, an edge function or an agent run SQL against a database over HTTP. It needs no database driver and holds no connection. **SQL stays the language.** The API carries SQL in the target's own dialect: PostgreSQL for `postgres:` targets, TiDB/MySQL for `sql:` targets, and DataFusion for `engine:` targets. It adds no query language of its own.

What the API adds around SQL:
- statement batches;
- named operations;
- opt-in interactive transactions;
- idempotency keys;
- typed, retry-aware errors;
- dry runs;
- budgets;
- three result encodings;
- live queries;
- a safety profile for agents.

### 1.1 Non-goals

- A new query language, or REST over tables (PostgREST-style).
- Replacing the Postgres or MySQL wire protocol. Drivers keep working beside this API.
- Client streaming or bidirectional streaming. Every call is unary or server-streaming (§3).
- Schema management and migrations. DDL may be sent as SQL, but the API adds nothing for it.

## 2. Conventions

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY and OPTIONAL are to be read as described in BCP 14 (RFC 2119, RFC 8174), and only when they appear in all capitals.

- **Server**: an implementation of `loams.data.v1.DataService`.
- **Target**: the database a request runs against (§5.1).
- **Batch**: an ordered list of statements in one `Execute` request.
- **Baton**: an opaque, single-use token that continues an interactive transaction (§4.3).
- **Protected branch**: a branch that the deployment marks as protected. A production branch is the usual example.
- **Agent credential**: a credential whose kind is `agent` (§12).

Durations default to the values given here. A deployment MAY configure other defaults, but it MUST NOT exceed a stated maximum.

## 3. Transports

- A server MUST serve [Connect](https://connectrpc.com/docs/protocol) unary calls over HTTP/1.1 and HTTP/2, with both the JSON (`application/json`) and protobuf (`application/proto`) codecs. A unary call is a single `POST /loams.data.v1.DataService/<Method>`, so `curl` and `fetch` work. RPCs marked `NO_SIDE_EFFECTS` MAY also be served over Connect `GET`.
- A server MUST serve the server-streaming RPCs (`ExecuteArrow`, `Subscribe`) over the Connect streaming protocol. It SHOULD also serve them over gRPC and gRPC-Web.
- A server MAY serve a **WebSocket** at `/loams.data.v1/ws` (subprotocol `loams.data.v1`). Each text frame is one JSON object:
  - from the client: `{"id": <string>, "method": "<Method>", "message": <request JSON>, "headers": {...}}`;
  - from the server: `{"id": <string>, "message": <response JSON>}`, `{"id": <string>, "error": <Connect error JSON>}` or, for a stream, `{"id": <string>, "end": true}`.

  Frames carry exactly the same messages as the Connect calls. They are not a gRPC stream. A client MAY have several requests in flight on one socket. The socket only lowers per-call latency; its semantics are the same as the HTTP calls. `headers` carries the per-request headers of §5.2 and §5.4.
- Servers MUST use TLS except on loopback.

## 4. Requests

### 4.1 Statement batch: `Execute`

- `statements` MUST hold 1–256 statements. Each statement is a single SQL statement. If `sql` holds more than one statement, the server MUST refuse it with `multiple_statements` (PERMANENT).
- A statement uses positional parameters or named parameters, not both (`invalid_argument`). Parameter values are encoded as in §9.1. `Param.type` MAY name the target type. Without it, the server lets the engine infer the type.
- `BATCH_MODE_TRANSACTION` (the default) runs the whole batch in one transaction. Either every statement commits or none does. If statement *i* fails, the server MUST roll back and return an error whose `statement_index` is *i*. The response has no partial results.
- In `BATCH_MODE_AUTOCOMMIT_EACH`, each statement commits on its own, and the batch stops at the first failure. The response is a success. `results` ends at the failed statement, and that result's `error` is set. Statements before it stay committed.
- The batch MUST NOT contain transaction control (`BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `SET TRANSACTION`). The server refuses it with `transaction_control_not_allowed`.
- If `read_only` is set, the server MUST run the batch in a read-only transaction.

### 4.2 Named operation: `Call`

`Call` invokes a registered operation (§10) with arguments by name. The rules of §5–§9 apply as they do for `Execute`.

### 4.3 Interactive transactions: `Begin`, `Continue`, `Commit`, `Rollback`

Interactive transactions are **opt-in**. Clients SHOULD use batches or named operations instead.

- `Begin` opens a transaction and returns a **baton**. Each later `Continue` returns a new baton. A baton is valid for exactly one call:
  - a server MUST refuse a baton that has already been used, with `baton_invalid` (PERMANENT);
  - a server MUST refuse an unknown baton the same way.

  A baton MUST be unguessable (at least 128 bits of entropy) and bound to the credential that called `Begin`.
- The `RequestContext` given to `Begin` (target, consistency, budget) holds for the whole transaction. `Continue`, `Commit` and `Rollback` carry only the baton.
- **Idle timeout.** If no call arrives within `idle_timeout`, the server MUST roll the transaction back. `idle_timeout` defaults to 10 s, and its maximum is 60 s. A later call with that baton fails with `baton_expired` (PERMANENT). A server MAY also cap a transaction's total lifetime. A cap of 5 min is RECOMMENDED.
- **Budget.** The budget applies to each `Continue` call separately (this draft's choice; see Appendix B).
- **Errors.** If a statement fails inside `Continue`, the server MUST roll the transaction back and return the error. The baton is then dead. Savepoint-based recovery is out of scope for v1.
- `Commit` MAY carry an `idempotency_key`. The server records it inside the transaction being committed (§6.1).
- `Rollback` is idempotent. An expired, used or unknown baton returns success.

A baton pins its transaction to one upstream connection on one server. Behind a load balancer, the baton MUST encode enough to route the call back to that server. How it does so is up to the implementation.

## 5. The request context

Every request that names a target carries a `RequestContext`.

### 5.1 Target

`target` is one of:

| Form | Meaning |
|---|---|
| `postgres:<project>/<branch>/<database>` | A PostgreSQL-compatible database on a branch |
| `sql:<database>/<branch>` | A MySQL-compatible (TiDB) database on a branch |
| `engine:<namespace>` | An analytical engine namespace (DataFusion SQL). It has no branches |

- Path segments MUST match `[A-Za-z0-9][A-Za-z0-9._/-]{0,127}`. For `postgres:` only, a branch name MAY contain `/` (as in `agent/s-123`). The last segment is then the database.
- An unknown or malformed target is `invalid_target` (PERMANENT). A target the credential cannot access is `permission_denied`. A server MUST NOT reveal whether such a target exists.
- `ResponseMeta.effective_target` reports where the request actually ran (§12.3).

### 5.2 Authentication

- Requests carry `Authorization: Bearer <credential>`. A server MUST reject an unauthenticated request with Connect code `unauthenticated` and reason `unauthenticated`.
- The server maps the credential to a database role. It MUST NOT run a request with more privilege than that role has. `TODO(Q917)`: the mapping. The design's default follows PostgREST: a per-project authenticator role, `SET LOCAL ROLE <mapped role>`, and the claims placed in `loams.claims` for row-level security.

### 5.3 Consistency

| Mode | Meaning |
|---|---|
| `STRONG` (default) | Read the primary's latest committed state |
| `AT_LEAST` + token | Read from any replica that has applied at least the state the token names. A server that cannot meet it within the budget's duration MUST fail with `consistency_unavailable` (RETRYABLE_UNAVAILABLE) |
| `REPLICA_OK` | Read from any replica. Staleness is bounded but not reported |

- Every successful response that commits a write MUST return a `consistency_token`. Read responses SHOULD return one.
- Tokens are opaque. A client MUST NOT parse them. A token is valid only for the target that issued it.
- Writes always run on the primary, whatever the mode.

### 5.4 Trace context and attributes

- Clients SHOULD send [W3C Trace Context](https://www.w3.org/TR/trace-context/) `traceparent` and `tracestate` headers. On the WebSocket they go in the frame's `headers`.
- A server MUST continue the trace in its own spans. It SHOULD pass the trace context to the engine, for example as a sqlcommenter comment (Loams D1038).
- `attributes` are key–value labels on the request, such as `app.tenant` or `app.user_agent`. A server MAY attach them to spans and pass them to the engine. It MUST NOT use them for authorisation. It MUST NOT put them on metric labels.

### 5.5 Version header

- A client MAY send `Loams-Data-Api-Version: 1.0`. A server that cannot serve that version MUST refuse with `unsupported_version` (PERMANENT).
- Every response's `ResponseMeta.api_version` names the version that was served.

## 6. Idempotency and errors

### 6.1 Idempotency keys

- `idempotency_key` is 16–64 bytes. Any other length is `invalid_argument`. Clients SHOULD use 16 random bytes.
- The **dedup window** is 24 hours from the first attempt's completion. Within the window, a request with the same key, from the same credential, on the same target:
  - with the **same request digest**, MUST NOT execute again. The server returns the stored result with `idempotent_replay = true`;
  - with a **different digest**, MUST be refused with `idempotency_key_reused` (PERMANENT).
- The **request digest** is a hash of the request with `idempotency_key`, the trace headers and `attributes` left out. Its exact form is implementation-defined, but it MUST be stable for at least the window.
- **Exactly once across a lost commit.** For `postgres:` and `sql:` writes, a server MUST record the key inside the transaction it protects, before `COMMIT`. The Loams implementation writes a row in `_loams.requests(key, digest, result, expires_at)`. The key column MUST be unique, so concurrent attempts with the same key serialise:
  - a second attempt that hits the unique violation MUST roll back and return the first attempt's stored result, once that has committed;
  - a retry after `COMMIT_UNKNOWN` finds the row, and so the stored result. If it finds no row, the first attempt did not commit, and the retry executes normally.
- **Stored results.** A server MUST store at least the `ResponseMeta` and the per-statement `rows_affected` and `command_tag`. It SHOULD store the full response up to 1 MiB. A replay of a larger result returns the rows only if they were stored. Otherwise `ResultSet.rows` is empty and `ResponseMeta.replay_truncated` is set (draft choice; see Appendix B).
- In `AUTOCOMMIT_EACH` mode, statement *i* is recorded under (key, *i*). A retry skips the statements already recorded.
- **Read-only requests** and `engine:` targets MAY deduplicate through a ledger outside the transaction. A server that cannot make writes exactly-once on a target MUST NOT claim the `exactly-once` conformance property for that target (§14).
- An operation whose rule is `IDEMPOTENCY_RULE_REQUIRED` refuses a call without a key with `idempotency_key_required`.

### 6.2 Error categories

Every error MUST carry exactly one `loams.data.v1.ErrorDetail` in the Connect error's `details`. The category tells the client what to do:

| Category | Meaning | Connect code | Client behaviour |
|---|---|---|---|
| `RETRYABLE_CONFLICT` | The transaction lost a conflict. Nothing was applied | `aborted` | Retry the whole batch, call or transaction, with backoff |
| `RETRYABLE_UNAVAILABLE` | The service could not run the request now. Nothing was applied | `unavailable` | Retry later, honouring `retry_after` |
| `COMMIT_UNKNOWN` | `COMMIT` was sent, and its outcome was lost | `unknown` | Retry **only** with the same idempotency key. Without a key, surface the error to the caller |
| `BUDGET_EXCEEDED` | A budget (§8) was hit. Nothing was applied, or it was rolled back | `resource_exhausted` | Do not retry unchanged. Raise the budget or narrow the query |
| `PERMANENT` | The request is wrong, or not allowed | `invalid_argument`, `permission_denied`, `not_found`, `failed_precondition`, `unauthenticated`, `unimplemented` | Do not retry |

- The category is authoritative. A client MUST decide retries from `category` and not from the Connect code.
- A server MUST NOT return `COMMIT_UNKNOWN` unless it actually sent a commit. That includes an autocommit statement that may write.
- A server SHOULD resolve the outcome itself before falling back to `COMMIT_UNKNOWN`, for example by checking the idempotency row on a new connection.
- SDKs SHOULD retry the `RETRYABLE_*` categories automatically, with jittered exponential backoff (at most 5 attempts is RECOMMENDED).
- SDKs MUST NOT retry `COMMIT_UNKNOWN` without an idempotency key, and MUST NOT retry `PERMANENT`.

### 6.3 Reasons

`reason` is a stable snake_case string. New reasons MAY be added in minor versions (§13), so clients MUST treat an unknown reason by its category.

| Reason | Category |
|---|---|
| `serialization_failure`, `deadlock`, `write_conflict`, `lock_not_available`, `schema_changed` | RETRYABLE_CONFLICT |
| `unavailable`, `compute_waking`, `too_many_connections`, `consistency_unavailable`, `admin_shutdown` | RETRYABLE_UNAVAILABLE |
| `commit_unknown` | COMMIT_UNKNOWN |
| `budget_duration`, `budget_rows`, `budget_bytes`, `budget_cost` | BUDGET_EXCEEDED |
| `invalid_argument`, `invalid_target`, `multiple_statements`, `transaction_control_not_allowed`, `syntax_error`, `undefined_object`, `permission_denied`, `unauthenticated`, `integrity_violation`, `data_exception`, `read_only_violation`, `protected_branch`, `raw_sql_write_not_allowed`, `result_too_large`, `idempotency_key_reused`, `idempotency_key_required`, `baton_invalid`, `baton_expired`, `operation_not_found`, `operation_nondeterministic`, `operation_changed`, `write_set_violation`, `subscription_too_large`, `subscription_limit`, `subscription_requires_read_only`, `unsupported_version`, `unimplemented` | PERMANENT |

`metadata` MAY carry details. Defined keys: `constraint`, `table`, `column`, `budget_field`, `limit`, `observed`.

### 6.4 Mapping from engine errors

A server MUST map engine errors as follows. A code not listed here is PERMANENT, with reason `invalid_argument` unless a closer reason fits.

| Engine signal | Category | Reason |
|---|---|---|
| SQLSTATE `40001` (serialization failure) | RETRYABLE_CONFLICT | `serialization_failure` |
| SQLSTATE `40P01` (deadlock) | RETRYABLE_CONFLICT | `deadlock` |
| SQLSTATE `55P03` (lock not available) | RETRYABLE_CONFLICT | `lock_not_available` |
| SQLSTATE `40003` (statement completion unknown) | COMMIT_UNKNOWN | `commit_unknown` |
| TiDB `9007` (write conflict); `8002`, `8022` (transaction retryable) (verify) | RETRYABLE_CONFLICT | `write_conflict` |
| TiDB `8028` (information schema changed) | RETRYABLE_CONFLICT | `schema_changed` |
| MySQL `1213` (deadlock), `1205` (lock wait timeout) (verify) | RETRYABLE_CONFLICT | `deadlock`, `lock_not_available` |
| SQLSTATE `57P01`–`57P03` (admin shutdown, crash shutdown, cannot connect now) | RETRYABLE_UNAVAILABLE | `admin_shutdown` / `unavailable` |
| SQLSTATE `53300` (too many connections) | RETRYABLE_UNAVAILABLE | `too_many_connections` |
| TiDB `9001`, `9002`, `9005` (PD, TiKV or region unavailable) (verify) | RETRYABLE_UNAVAILABLE | `unavailable` |
| A compute that is starting (scale to zero) | RETRYABLE_UNAVAILABLE | `compute_waking` |
| Connection failure (`08xxx`, reset, timeout) **before** `COMMIT` was sent | RETRYABLE_UNAVAILABLE | `unavailable` |
| Connection failure **after** `COMMIT` was sent, or during an autocommit write | COMMIT_UNKNOWN | `commit_unknown` |
| SQLSTATE `57014` (query canceled) raised by the budget timer | BUDGET_EXCEEDED | `budget_duration` |
| SQLSTATE `57014` from a client cancel | (Connect `canceled`) | `canceled` |
| SQLSTATE `25006` (read-only transaction) | PERMANENT | `read_only_violation` |
| SQLSTATE class `23` (integrity constraint) | PERMANENT | `integrity_violation` |
| SQLSTATE class `42` (syntax, undefined object, privilege) | PERMANENT | `syntax_error`, `undefined_object`, `permission_denied` |
| SQLSTATE class `22` (data exception) | PERMANENT | `data_exception` |

`sqlstate` and `engine_code` MUST carry the original codes.

## 7. Dry run: `Explain`

`Explain` executes nothing and commits nothing. For a batch, or for a call, it returns:
- per statement: the `kind`, the engine's plan (`plan_json`, the engine's native JSON format), `estimated_rows`, `estimated_cost_units`, the `query_id`, and the `applied_pins` (the plan pins that would apply);
- the **write set**: each table that would be written, the kind of change (`INSERT`, `UPDATE`, `DELETE`) and the estimated rows. For a named operation, it is the operation's declared write set, with `estimated_rows = -1` where the server cannot estimate;
- `over_budget`: the budget reasons (§6.3) that the estimates already exceed.

Rules:
- A server MUST NOT apply any effect. For `postgres:`, it runs `EXPLAIN (FORMAT JSON)` without `ANALYZE`, inside a transaction that is always rolled back.
- **Utility statements** (DDL, `SET` and so on) are reported as `STATEMENT_KIND_UTILITY`, with no plan, and are not executed.
- A statement that cannot be planned gets its own `error`, and the other statements are still planned. A statement that depends on an object created earlier in the same batch therefore reports `undefined_object`.
- `Explain` is `NO_SIDE_EFFECTS`. Agent credentials MAY always call it, subject to their read permissions.

## 8. Budgets

### 8.1 Fields and enforcement

`Budget { max_duration, max_rows, max_bytes, max_cost_units }`. A zero field means the credential's default. A request MAY lower a cap below the credential's maximum. A request that asks for more is clamped to the maximum, silently, and the response reports what was used in `usage`.

| Field | Enforcement | On breach |
|---|---|---|
| `max_duration` | A per-transaction statement timeout (`SET LOCAL statement_timeout` on Postgres), plus a server-side deadline | `budget_duration` |
| `max_rows` | Rows returned plus rows written, counted while the call runs | `budget_rows` |
| `max_bytes` | Result bytes in the response encoding, counted while streaming | `budget_bytes` |
| `max_cost_units` | Checked at planning, **before execution** | `budget_cost` |

- If a call hits a cap while running, the server MUST cancel the statement, roll back anything written, and return BUDGET_EXCEEDED.
- A server MUST NOT return a truncated result as a success.
- In `AUTOCOMMIT_EACH` mode, statements already committed stay committed. The failing statement's result then carries the error (§4.1).

### 8.2 Cost units

Cost units are the planner's total cost estimate on `postgres:` targets, and estimated rows × estimated row width on `sql:` and `engine:` targets. They are comparable only within one target kind (informative: this is a known wart; see Appendix B).

### 8.3 Defaults

| Credential | `max_duration` | `max_rows` | `max_bytes` | `max_cost_units` |
|---|---|---|---|---|
| Agent (§12) | 5 s | 10 000 | 16 MiB | deployment-set |
| Others | deployment-set | deployment-set | deployment-set | deployment-set |

A server MUST apply a budget to every request. "Unlimited" is not a valid default.

## 9. Results

### 9.1 JSON rows (`Execute`, `Call`, `Subscribe`)

- A result set is `columns` plus `rows`. Each row is a JSON array with one element per column.
- The column schema gives: `name`, the engine `type` name, `type_oid` (Postgres; 0 otherwise), `nullable`, `origin_table` and `origin_column` where the column comes straight from a table, and `is_key` where it belongs to that table's primary key.
- **Value encoding** (for both parameters and rows):

| SQL type | JSON |
|---|---|
| NULL | `null` |
| boolean | `true` / `false` |
| int2, int4, float4, float8 (finite) | number |
| int8, numeric/decimal, `NaN`/`±Infinity` floats | string (decimal text, or `"NaN"`, `"Infinity"`, `"-Infinity"`), so no precision is lost |
| text, varchar, char, uuid, enum | string |
| bytea / binary | string, standard base64 with padding |
| date, time, timestamp, timestamptz, interval | string, in the engine's ISO 8601 text form (timestamptz in UTC with `Z`) |
| json, jsonb | the JSON value itself |
| arrays | JSON array of the element encoding |
| anything else | string, in the engine's text output form |

- The JSON limits are 10 000 rows and 1 MiB per response, by default. A larger result fails with `result_too_large` (PERMANENT), and the client should use `ExecuteArrow` instead.
- With the protobuf codec, rows are the same `google.protobuf.ListValue`s.

### 9.2 Arrow IPC (`ExecuteArrow`)

- The response is a server stream. For each result set in the batch it carries one `ArrowHeader`, then either `ipc` chunks or one `partitions` manifest. One `ArrowTrailer` ends the stream.
- The `ipc` chunks of one result set, concatenated, MUST form exactly one valid [Arrow IPC stream](https://arrow.apache.org/docs/format/Columnar.html#ipc-streaming-format), schema message first.
- Arrow field metadata MUST carry `loams.type` (the engine type name). For Postgres it MUST also carry `loams.type_oid`.
- The inline limit is 256 MiB by default. `DELIVERY_AUTO` switches to partitions above it. `DELIVERY_INLINE` fails with `result_too_large`.
- The Arrow type mapping follows the Arrow types that the engine's own Arrow export uses. int8 is `Int64`, numeric is `Decimal128` or `Decimal256` (or `Utf8` when precision is unbounded), timestamptz is `Timestamp(us, "UTC")`, and json/jsonb is `Utf8` with the `arrow.json` extension name. `TODO`: a full table, aligned with the Loams events-to-Arrow rules.

### 9.3 Presigned partitions

- For very large results, the server writes the result as Parquet (the default) or Arrow IPC files, and returns a `Partitions` manifest. The manifest holds the URLs, row and byte counts, and `expires_at`. Clients fetch the parts with plain HTTP GET, in parallel.
- URLs MUST expire, after 1 h by default. The objects MUST be deleted no later than 24 h after creation.
- A URL MUST grant read access to that one object and nothing else.
- Where the object store cannot presign, the URL MAY point at a server download endpoint that requires the same credential.
- `TODO(Q918)`: object location (the Loams default is `ns/<ns>/results/<request_id>/part-<n>.parquet`) and the threshold.

## 10. Named operations

A named operation is **a SQL function in the database**, registered with the Data API. It lives with the schema, versions with migrations, branches with the database, and runs under the database's own privilege checks.

### 10.1 Registration (`OperationService`)

- `RegisterOperation(target, function, ...)` registers a function. The function is a Postgres `CREATE FUNCTION` (by default in schema `api`), a stored routine on `sql:` targets, or a registered parameterised statement where the engine has no functions (verify for TiDB).
- The server records:
  - the parameter and result types, from the catalog;
  - the **declared write set**;
  - the idempotency rule;
  - the allowed callers;
  - whether agents may call it;
  - a description (the function's `COMMENT` by default);
  - a JSON Schema for the arguments;
  - the `definition_digest`.
- If the function's definition later differs from `definition_digest`, calls MUST fail with `operation_changed` until the operation is registered again.
- An empty write set declares the operation read-only. The server MUST then run it in a read-only transaction.
- The server MUST enforce the write set at execution, so that a write outside it fails with `write_set_violation`. The RECOMMENDED way is to run the operation as a role whose write grants are exactly the write set.

### 10.2 Determinism

- An operation MUST be deterministic given the database state and its arguments. Registration MUST refuse, with `operation_nondeterministic`, a function whose body calls the volatile clock or randomness functions directly: `now()`, `current_timestamp`, `clock_timestamp()`, `statement_timestamp()`, `random()`, `gen_random_uuid()`, `uuid_generate_*()`, and the target dialect's equivalents. `TODO`: the check is a parse of the function body; how deep it goes into called functions is unspecified in v0.
- Instead, an operation that needs time or randomness declares two trailing parameters. The server injects them, and callers MUST NOT pass them:
  - `_loams_now timestamptz`: the server's clock at call time;
  - `_loams_seed bytea`: 32 random bytes.
- The server SHOULD provide `loams.uuid(seed bytea, n int) → uuid`, a deterministic UUIDv4-shaped derivation, so operations can mint ids.
- Determinism here means "same state + same arguments + same injected values ⇒ same writes and result". It does not cover concurrency. Replay (§10.4) reproduces a call only against the same starting state and in the same order.

### 10.3 Calls

- `Call` runs the operation in its own transaction, under the caller's role (§5.2), and checks the caller against `allowed_callers` first.
- The response carries the result, the `call_id`, the injected `_loams_now` and `_loams_seed`, and `result_digest`, the SHA-256 of the result's canonical JSON encoding: §9.1 values, no whitespace, UTF-8.

### 10.4 Call log and replay

- Every committed call that writes MUST be logged as a [CloudEvent](https://cloudevents.io) of type `io.loams.dev.data.call.v1`. Its data holds:
  - `target`, `operation` and `definition_digest`;
  - `args`, `now` and `seed`;
  - `result_digest`, `call_id` and the commit's `consistency_token`.

  Read-only calls MAY be logged.
- The log MAY hold argument values that are sensitive. Its access control MUST be at least as strict as the target's.
- `ReplayCalls` re-runs logged calls, in the given order, on a non-protected target branch. It uses the logged arguments and injected values, and reports for each call whether `result_digest` matched. This is how an agent's changes are tested on a fresh branch before a human merges them.

## 11. Live queries: `Subscribe`

### 11.1 Model

- `Subscribe` takes one read-only statement or one read-only operation. A query that writes is refused with `subscription_requires_read_only`.
- The stream starts with a `Snapshot` (the full result and a cursor). After that it carries `Diff` events (`added`, `removed`, `changed`, each with a cursor), and a `Heartbeat` at least every 30 s while nothing changes.
- **Keys.** If the result's `is_key` columns form a key of every origin table in the result, the server MUST key rows by them. `changed` then carries a row whose key stayed the same and whose other values differ. Otherwise the server keys rows by their full value, treating the result as a multiset, and never emits `changed`.
- **Mechanism.** The server finds the relations the query reads, and follows their changes on the target's change stream:
  - logical decoding (pgoutput) for `postgres:`;
  - CDC for `sql:`;
  - changelog streams for `engine:`.

  When a relevant change arrives, the server re-runs the query, debounced by `debounce` (100 ms by default), and sends the difference. `TODO(Q919)`: incremental view maintenance instead of re-running.
- Results are delivered at the subscription's consistency (§5.3). Each diff reflects a committed state no older than the change that caused it.

### 11.2 Resume and limits

- A client that reconnects with `cursor` gets the diffs since that cursor. If the change stream has been trimmed past it, the client instead gets a `Snapshot` with `reset = true`.
- There is a per-credential cap on concurrent subscriptions, 100 by default (`subscription_limit`).
- There is a per-subscription result cap, 10 000 rows by default (`subscription_too_large`). The subscription's budget applies to each re-run.

### 11.3 CloudEvents mapping

Each `SubscribeResponse` maps to a CloudEvent:
- `id` = `event_id`;
- `source` = `/loams/data/v1/<target>`;
- `type` = `io.loams.dev.data.subscription.{snapshot|diff|heartbeat}.v1`;
- `time` = `time`;
- `data` = the event message in its JSON form.

A server MAY offer the same stream as CloudEvents in structured mode on the WebSocket.

## 12. The agent safety profile

The profile is the default for agent credentials. A deployment MUST NOT turn it off for agent credentials, but MAY tighten it.

### 12.1 Read-only by default

Agents' raw SQL (`Execute`, `ExecuteArrow`, `Begin`, `Subscribe`, and the MCP `query` tool) MUST run read-only. That means a read-only transaction **and** a role without write grants. Raw SQL that writes is refused with `raw_sql_write_not_allowed`, even on an agent branch.

### 12.2 Writes only through named operations

An agent writes only by calling an operation registered with `agent_callable = true`. All other operations are refused to agents with `permission_denied`.

### 12.3 Writes only on an agent branch

Answered by Q920 (owner, 2026-10-16): automatic per-session branches.
- On an agent's first write against a target, the server creates the branch `agent/<session>` from the target branch, and runs the write there. `<session>` is the agent session id from the credential, or the MCP session id.
- From then on, every request of that session against the same base target, reads included, runs on the agent branch, so the agent reads its own writes. `effective_target` names that branch.
- The caller MAY instead name a non-protected branch as its target, and writes then go there.
- An agent write MUST NOT reach a protected branch, whatever target is named.
- Agent branches expire **7 days** after their last write (this draft reads "TTL 7 days" as an idle TTL; see Appendix B). Expiry deletes the branch.
- Changes reach the real branch only through a human-approved merge, or through `ReplayCalls` (§10.4) run by a non-agent credential.
- `engine:` targets have no branches. Agents are read-only there in v1.

### 12.4 Budgets and MCP tools

- The budget defaults of §8.3 apply.
- A server that offers MCP (Model Context Protocol) generates the tools from the schema:
  - one tool per agent-callable operation. Its name is `<schema>_<function>`, its `inputSchema` is the operation's `args_json_schema`, and its description is the operation's description;
  - `describe_schema`: the tables, columns, keys and comments visible to the role;
  - `explain`: §7;
  - `query`: read-only raw SQL (§12.1).
- **Untrusted data.** Tool results MUST wrap row data in a typed envelope, `{"type": "loams.data.untrusted", "columns": [...], "rows": [...]}`. Tool descriptions MUST tell the model that row data is untrusted, and that instructions inside it are not to be followed.
- Tool annotations MUST mark `query`, `describe_schema` and `explain` as read-only. Operation tools are marked by their write set.

(Informative.) The profile removes each leg of the "lethal trifecta". Data stays read-only, the role is scoped, and writes are named, branched and budgeted.

## 13. Versioning

- The spec is versioned `data-api-MAJOR.MINOR`. This draft targets `data-api-1.0`. The proto package `loams.data.v1` covers every `1.x`.
- A **minor** version only adds things: fields, enum values, reasons, RPCs, and optional behaviour. Clients MUST ignore unknown fields, and MUST treat unknown enum values and reasons by their category.
- A **major** version gets a new proto package (`loams.data.v2`). A server SHOULD serve the previous major for at least one minor release after the new one ships.
- Draft versions (`data-api-1.0-draft.N`) carry no compatibility promise.

## 14. Conformance

An implementation claims one or more levels per target kind, for example "`data-api-1.0` Core + Transactions on `postgres:`". Each level requires the ones above it in this table, except Core, which every claim needs.

| Level | Requires |
|---|---|
| **Core** | §3 Connect unary; §4.1 `Execute` (both modes); §5; §6 (errors, the mapping, and idempotency keys); §7 `Explain`; §8 budgets; §9.1 JSON results; §13 |
| **Transactions** | §4.3 batons |
| **Arrow** | §9.2 `ExecuteArrow`; §9.3 partitions |
| **Operations** | §10 `OperationService`, `Call`, determinism, call log and replay |
| **Live** | §11 `Subscribe` |
| **Agent** | §12 the agent profile, including MCP tool generation. Requires Operations |
| **WebSocket** | §3 the WebSocket transport (optional) |

Properties claimed beside the levels:
- `exactly-once`: §6.1 in-transaction keys hold on this target.
- `branches`: the target kind has branches (needed for §12.3 writes).

The conformance suite (`conformance/`) is public, and passing it is how an implementation substantiates a claim.

---

## Appendix A. Prior art (informative)

- **Neon serverless driver.** SQL over HTTP for one-shot queries and over WebSocket for sessions and interactive transactions. It is the model for §3's two transports.
- **libSQL Hrana (HTTP v2).** Pipelined requests, with a baton that changes on every response to carry stream state between stateless HTTP calls. It is the model for §4.3.
- **Cloudflare D1 batches.** A statement list run as one implicit transaction. It is the model for §4.1's default mode.
- **Amazon Aurora DSQL.** Optimistic concurrency that reports conflicts at commit as retryable errors, which clients must retry whole. It motivates the RETRYABLE_CONFLICT category and the SDK retry rule.
- **PlanetScale traffic control.** Budgets per query, not only per connection (§8).
- **Convex mutations, Zero custom mutators, SpacetimeDB reducers.** Named, typed, deterministic server-side writes. §10 provides the same as standard SQL functions, with an injected clock and seed.
- **Stripe idempotency keys.** Keys with a 24-hour window, and refusal of a reused key with different parameters (§6.1). §6.1 goes further by recording the key inside the protected transaction.
- **PostgREST.** The authenticator-role pattern of §5.2 (Q917).
- **The Supabase MCP incident (2025).** It motivates §12.

## Appendix B. Choices this draft made where design 58 was silent

| Point | Choice | Revisit |
|---|---|---|
| Name of the dry-run RPC | `Explain` (design 58 D1043/D1045); "dry run" names the concept | Rename to `DryRun` only if reviewers prefer it |
| Registration surface | A separate `OperationService`, so it can be authorised as administration | — |
| `ExecuteArrow` shape | Server-streaming chunks, not one unary message of up to 256 MiB | — |
| JSON row shape | `ListValue` arrays, with 64-bit ints and decimals as strings | — |
| Budget inside interactive transactions | Per `Continue` call; the lifetime is capped separately | new question |
| Agent branch TTL | 7 days since the last write, not since creation | Q920 follow-up |
| Agent reads after the first write | Redirected to the agent branch (read your own writes) | — |
| Replay of results over 1 MiB | Counts only, flagged as truncated | — |
| Keys for diffs without a primary key | Multiset by full row; no `changed` | Q919 |
| Cost units across target kinds | Not comparable | — |
| Error detail type | Its own `loams.data.v1.ErrorDetail`, so the spec repository is self-contained. Loams maps `reason` to its `loams.errors.v1.ErrorInfo` registry | — |

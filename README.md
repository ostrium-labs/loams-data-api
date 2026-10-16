# Loams Data API

An open specification for running SQL over HTTP: statement batches, named operations, opt-in interactive transactions, idempotency keys, typed retry-aware errors, dry runs, budgets, JSON, Arrow and presigned-Parquet results, live queries, and a safety profile for AI agents.

**Status: draft v0.** Nothing is stable yet. The first stable version will be `data-api-1.0`.

## What is here

| Path | What |
|---|---|
| [`spec/data-api.md`](spec/data-api.md) | The normative specification (RFC 2119 language) |
| [`proto/loams/data/v1/data.proto`](proto/loams/data/v1/data.proto) | The wire contract: the Connect/gRPC services `DataService` and `OperationService`, package `loams.data.v1` |
| [`conformance/`](conformance/README.md) | The planned conformance suite: its layout and example cases |
| `buf.yaml` | `buf lint` configuration (STANDARD and COMMENTS rules) |

## Scope

- **SQL is the language.** Each target uses its own dialect: PostgreSQL, TiDB/MySQL or DataFusion. The API carries SQL and adds no query language of its own.
- **Three target kinds:** `postgres:<project>/<branch>/<database>`, `sql:<database>/<branch>` and `engine:<namespace>`.
- **Transports:** Connect over HTTP/1.1 and HTTP/2 (JSON or protobuf), so `curl` and `fetch` work from edge runtimes. An optional WebSocket carries the same messages.
- **Implementations:** Loams implements the API in `loams-gateway` (for Postgres and Loams SQL) and in the `loams` binary (for engine SQL). Any other database may implement it and claim conformance levels (spec §14).

## Non-goals

- A new query language, or a REST-over-tables API in the style of PostgREST.
- Replacing the Postgres or MySQL wire protocols. Drivers keep working beside this API.
- Client streaming or bidirectional streaming RPCs.
- Schema migration tooling.

## Relation to Loams

The spec comes from Loams design 58 §5 (decisions D1042–D1050). Loams mirrors `proto/loams/data/v1/` into its own `proto/` tree unchanged. Open points are marked `TODO(Qnnn)` with the design's question numbers.

## Checking the protos

```sh
buf lint
buf build
```

## Licence

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE). Contributions are accepted under the same licence, with a DCO sign-off (`git commit -s`).

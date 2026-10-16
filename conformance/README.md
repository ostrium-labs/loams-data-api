# Conformance suite (planned)

**Status: layout and examples only.** The runner does not exist yet. The suite checks an implementation against [`spec/data-api.md`](../spec/data-api.md), one level at a time (spec §14).

## Layout

```
conformance/
  README.md
  fixtures/<dialect>/<name>.sql     setup SQL, run before a case on a fresh database or branch
  cases/<level>/<id>.yaml           one case per file
  runner/                           (planned) a small runner that speaks Connect JSON
```

`<level>` is one of `core`, `transactions`, `arrow`, `operations`, `live`, `agent` or `websocket`. `<dialect>` is `postgres`, `sql` or `engine`.

## Case format

Every case is a YAML file:

```yaml
id: core-001                 # unique; matches the file name
title: one line
spec: "§4.1"                 # the clause under test
level: core
targets: [postgres, sql]     # target kinds the case applies to
requires: []                 # properties, e.g. [exactly-once], [branches]
fixture: accounts            # fixtures/<dialect>/accounts.sql
credential: app              # app | agent | admin (the runner's configured credentials)
fault: none                  # optional fault injection, see below
steps:
  - call: Execute            # an RPC of DataService or OperationService
    request: { ... }         # the Connect JSON request; "$target" is substituted
    expect:
      class: ok              # ok | error
      category: ~            # for errors: RETRYABLE_CONFLICT, ... (spec §6.2)
      reason: ~              # for errors: the §6.3 reason
      assert: [ ... ]        # optional JSONPath assertions over the response
    save: { name: "$.path" } # optional: values for later steps ("${name}")
```

- The **expected response class** is the main assertion: `ok`, or `error` with its `category` and `reason`.
- A case passes only if every step matches.
- Messages and plan text are never asserted on.
- `assert` entries are `{path, equals | matches | length}`.

### Fault injection

`COMMIT_UNKNOWN` and conflicts cannot be tested without help from the server. A server in **conformance mode** honours the request header `Loams-Conformance-Fault`. It MUST refuse that header otherwise. Defined faults:
- `drop-commit-ack`: commit, then fail as if the acknowledgement was lost;
- `drop-before-commit`: fail before sending `COMMIT`;
- `conflict`: fail the transaction with a serialization failure.

## Example cases

These are the first ten, written out here until the files exist. Each one would become `cases/<level>/<id>.yaml`.

```yaml
# cases/core/core-001.yaml
id: core-001
title: A batch is one transaction by default; a failure rolls back every statement
spec: "§4.1"
level: core
targets: [postgres, sql]
fixture: accounts            # accounts(id int primary key, balance int)
credential: app
steps:
  - call: Execute
    request:
      context: { target: "$target" }
      statements:
        - { sql: "INSERT INTO accounts VALUES (10, 5)" }
        - { sql: "INSERT INTO accounts VALUES (10, 6)" }   # duplicate key
    expect: { class: error, category: ERROR_CATEGORY_PERMANENT, reason: integrity_violation,
              assert: [ { path: "$.details[0].debug.statementIndex", equals: 1 } ] }
  - call: Execute
    request: { context: { target: "$target" }, statements: [ { sql: "SELECT count(*) FROM accounts WHERE id = 10" } ] }
    expect: { class: ok, assert: [ { path: "$.results[0].result.rows[0][0]", equals: "0" } ] }
```

```yaml
# cases/core/core-002.yaml
id: core-002
title: AUTOCOMMIT_EACH keeps earlier statements and reports the failure in-band
spec: "§4.1"
level: core
targets: [postgres, sql]
fixture: accounts
credential: app
steps:
  - call: Execute
    request:
      context: { target: "$target" }
      mode: BATCH_MODE_AUTOCOMMIT_EACH
      statements:
        - { sql: "INSERT INTO accounts VALUES (20, 1)" }
        - { sql: "INSERT INTO accounts VALUES (20, 2)" }
        - { sql: "INSERT INTO accounts VALUES (21, 3)" }
    expect:
      class: ok
      assert:
        - { path: "$.results", length: 2 }
        - { path: "$.results[1].error.reason", equals: integrity_violation }
```

```yaml
# cases/core/core-003.yaml
id: core-003
title: int8 and numeric values are JSON strings; the schema reports key columns
spec: "§9.1"
level: core
targets: [postgres]
fixture: accounts
credential: app
steps:
  - call: Execute
    request:
      context: { target: "$target" }
      statements: [ { sql: "SELECT id, 9007199254740993::int8 AS big, 1.10::numeric AS n FROM accounts WHERE id = 1" } ]
    expect:
      class: ok
      assert:
        - { path: "$.results[0].result.rows[0][1]", equals: "9007199254740993" }
        - { path: "$.results[0].result.rows[0][2]", equals: "1.10" }
        - { path: "$.results[0].result.columns[0].isKey", equals: true }
```

```yaml
# cases/core/core-004.yaml
id: core-004
title: A retried idempotency key returns the stored result and does not write twice
spec: "§6.1"
level: core
targets: [postgres, sql]
requires: [exactly-once]
fixture: accounts
credential: app
steps:
  - call: Execute
    request: &req
      context: { target: "$target" }
      idempotencyKey: "AAECAwQFBgcICQoLDA0ODw=="      # 16 bytes, base64
      statements: [ { sql: "UPDATE accounts SET balance = balance + 1 WHERE id = 1" } ]
    expect: { class: ok }
  - call: Execute
    request: *req
    expect: { class: ok, assert: [ { path: "$.meta.idempotentReplay", equals: true } ] }
  - call: Execute
    request: { context: { target: "$target" }, statements: [ { sql: "SELECT balance FROM accounts WHERE id = 1" } ] }
    expect: { class: ok, assert: [ { path: "$.results[0].result.rows[0][0]", equals: 101 } ] }  # fixture balance 100
```

```yaml
# cases/core/core-005.yaml
id: core-005
title: The same key with a different request is refused
spec: "§6.1"
level: core
targets: [postgres, sql, engine]
fixture: accounts
credential: app
steps:
  - call: Execute
    request: { context: { target: "$target" }, idempotencyKey: "EBESExQVFhcYGRobHB0eHw==",
               statements: [ { sql: "SELECT 1" } ] }
    expect: { class: ok }
  - call: Execute
    request: { context: { target: "$target" }, idempotencyKey: "EBESExQVFhcYGRobHB0eHw==",
               statements: [ { sql: "SELECT 2" } ] }
    expect: { class: error, category: ERROR_CATEGORY_PERMANENT, reason: idempotency_key_reused }
```

```yaml
# cases/core/core-006.yaml
id: core-006
title: A lost commit acknowledgement is COMMIT_UNKNOWN, and a keyed retry resolves it
spec: "§6.1, §6.4"
level: core
targets: [postgres, sql]
requires: [exactly-once]
fixture: accounts
credential: app
fault: drop-commit-ack       # applies to the first step only
steps:
  - call: Execute
    request: &req
      context: { target: "$target" }
      idempotencyKey: "ICEiIyQlJicoKSorLC0uLw=="
      statements: [ { sql: "INSERT INTO accounts VALUES (30, 0)" } ]
    expect: { class: error, category: ERROR_CATEGORY_COMMIT_UNKNOWN, reason: commit_unknown }
  - call: Execute
    request: *req
    expect: { class: ok, assert: [ { path: "$.meta.idempotentReplay", equals: true } ] }
```

```yaml
# cases/core/core-007.yaml
id: core-007
title: Explain returns the write set and executes nothing
spec: "§7"
level: core
targets: [postgres]
fixture: accounts
credential: app
steps:
  - call: Explain
    request:
      context: { target: "$target" }
      statements: { statements: [ { sql: "DELETE FROM accounts WHERE balance < 0" } ] }
    expect:
      class: ok
      assert:
        - { path: "$.plans[0].kind", equals: STATEMENT_KIND_WRITE }
        - { path: "$.writeSet[0].table", equals: "public.accounts" }
        - { path: "$.writeSet[0].kind", equals: WRITE_KIND_DELETE }
```

```yaml
# cases/core/core-008.yaml
id: core-008
title: A row budget is enforced; the result is not truncated into a success
spec: "§8.1"
level: core
targets: [postgres, sql, engine]
fixture: series
credential: app
steps:
  - call: Execute
    request:
      context: { target: "$target", budget: { maxRows: "10" } }
      statements: [ { sql: "SELECT * FROM series" } ]          # 1 000 rows
    expect: { class: error, category: ERROR_CATEGORY_BUDGET_EXCEEDED, reason: budget_rows }
```

```yaml
# cases/transactions/tx-001.yaml
id: tx-001
title: A baton is single-use, and an expired transaction is rolled back
spec: "§4.3"
level: transactions
targets: [postgres, sql]
fixture: accounts
credential: app
steps:
  - call: Begin
    request: { context: { target: "$target" }, idleTimeout: "1s" }
    expect: { class: ok }
    save: { b0: "$.baton" }
  - call: Continue
    request: { baton: "${b0}", statements: [ { sql: "INSERT INTO accounts VALUES (40, 0)" } ] }
    expect: { class: ok }
  - call: Continue
    request: { baton: "${b0}", statements: [ { sql: "SELECT 1" } ] }   # reused baton
    expect: { class: error, category: ERROR_CATEGORY_PERMANENT, reason: baton_invalid }
  - wait: 2s
  - call: Execute
    request: { context: { target: "$target" }, statements: [ { sql: "SELECT count(*) FROM accounts WHERE id = 40" } ] }
    expect: { class: ok, assert: [ { path: "$.results[0].result.rows[0][0]", equals: "0" } ] }
```

```yaml
# cases/agent/agent-001.yaml
id: agent-001
title: An agent cannot write raw SQL; a named write goes to agent/<session>
spec: "§12.1, §12.3"
level: agent
targets: [postgres]
requires: [branches]
fixture: accounts_with_ops   # registers api.credit(id int, amount int) agent_callable, writes accounts
credential: agent            # agent session id "s-conf-1"; target branch "main" is protected
steps:
  - call: Execute
    request: { context: { target: "$target" }, statements: [ { sql: "UPDATE accounts SET balance = 0" } ] }
    expect: { class: error, category: ERROR_CATEGORY_PERMANENT, reason: raw_sql_write_not_allowed }
  - call: Call
    request: { context: { target: "$target" }, call: { operation: "api.credit", args: { id: 1, amount: 5 } } }
    expect:
      class: ok
      assert: [ { path: "$.meta.effectiveTarget", matches: "^postgres:[^/]+/agent/s-conf-1/[^/]+$" } ]
  - call: Execute
    credential: app          # the protected branch is unchanged
    request: { context: { target: "$target" }, statements: [ { sql: "SELECT balance FROM accounts WHERE id = 1" } ] }
    expect: { class: ok, assert: [ { path: "$.results[0].result.rows[0][0]", equals: 100 } ] }
```

## Still to write

- The cases for Arrow (`ipc` concatenation into a valid stream, the partition manifest, URL expiry), Operations (the nondeterminism rejection, injected `_loams_now`/`_loams_seed`, replay digests, `operation_changed`), Live (a snapshot then a diff, resume with a cursor, `reset` after a trim), and the SQLSTATE mapping table in §6.4. The mapping needs one case per row, using `fault: conflict` where the engine cannot be made to fail on demand.
- The runner, and the claim format an implementation publishes with its results.

# lib-http-middleware

`http-middleware` — onion-style request pipeline for handl HTTP
servers (P-007 / T-120). Sits between `serve`/`serve-epoll` (which hand
the handler a parsed `lib-http-rfc` request record and expect a
response record back) and `http-route`'s router. Middleware wraps the
router: inspect/transform the request, short-circuit with its own
response (auth 401, CORS preflight, body-limit 413), or post-process
the response on the way out.

## Layout

| path | role |
| --- | --- |
| `src/index.hnl` | the library — `chain`/`pipeline` folding + built-ins (CORS, logger, request-id, session, auth, CSRF, body-limit) |
| `src/index.test.hnl` | tests |
| `pkg.hnl` | manifest — dep `route` (`branch:main`) |
| `scripts/hnl-gate.sh`, `ci.sh` | gate + bupha entry |
| `workflows/gate.yaml` | CI — shared `handl-gates` campaign |

## Contract

- Protocol: `ctx := { req, state, ... }`, `next := (ctx) ~> { resp,
  state }`, middleware `(ctx, next) ~> { resp, state }`.
- Pure `.hnl`, no Rust crate.

## Verification

```bash
HNL=../host-cli/target/debug/hnl scripts/hnl-gate.sh
```

<!-- myco:agent-awareness -->

## Agent awareness

- Session start / any work discussion: run `myco status --prompt`
  before proposing next steps — know the board first.
- User describes new work → offer to capture it (`myco todo add`,
  `issue new`, `proposal new`). Don't wait to be asked.
- Picking up a task: `myco resume <ID>` is the briefing.
- Wrapping up: `myco handoff <ID>` anything resumable; keep statuses
  honest (claimed → in-progress → blocked/done).

<!-- /myco:agent-awareness -->

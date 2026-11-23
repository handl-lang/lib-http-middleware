# lib-http-middleware

`http-middleware` — onion-style request pipeline for handl HTTP
servers (proposal-0007 / T-120). Sits between `serve`/`serve-epoll`
(which hand the handler a parsed lib-http-rfc request record
`{ method, target, path, query, version, headers, body, body_bytes }`
and expect a response record `{ status?, headers?, body? | body_bytes? }`
back) and `http-route`'s router. Middleware wraps the router:
inspect/transform the request, short-circuit with its own response
(auth 401, CORS preflight, body-limit 413), or post-process the
response on the way out.

## Protocol

```text
ctx      := { req: Request, state: State, ... }   -- explicit record,
                                             -- threaded, no cells
next     := (ctx) ~> { resp, state }             -- the rest of the chain
mw       := (ctx, next) ~> { resp, state }       -- a middleware
terminal := (ctx) ~> resp | { resp, state }      -- LAST step; the router
```

State merges **right-wins**: a middleware passes state inward with
`hash_map.merge(ctx.state, additions)` (its keys win over what it
received) and returns the outward state in the `{ resp, state }` pair.
Returning a bare response record is allowed — it normalizes to
`{ resp, state: <incoming ctx.state> }`.

`chain([mw1, mw2, ..., terminal])` folds right-to-left into one
serve-ready `(req) ~> resp` — `serve(chain([...]), { port })` works
directly. `pipeline(steps)` exposes the same fold as
`(ctx) ~> { resp, state }` for callers that want to seed state or keep
the outward state. `end(h)` adapts a `(req) ~> resp` handler (e.g.
`http-route`'s `router(routes)` output) into a `(ctx) ~> resp`
terminal.

```handl
from "http-middleware" import * as mw
from "http-route" import * as rt
from "serve-epoll" import { serve }

app := rt.router(rt.route({
  ".":     index_h,
  health:  { get: health_h },
  secret:  authed_h
}))

serve(mw.chain([
  mw.cors({ origin: "https://app.example.com", credentials: true }),
  mw.request_id({}),
  mw.session(store),
  mw.auth(verify),
  mw.csrf(check),
  mw.body_limit(1 << 20),
  mw.end(app)
]), { port: 8080 })
```

Order matters: `session` before `auth` lets `verify(ctx)` read
`state_get(ctx, "session")`; `auth` before `csrf` rejects unauthenticated
mutations early.

## v1 middleware

| factory | does |
|---|---|
| `cors(opts)` | `Access-Control-*` headers; OPTIONS + `access-control-request-method` preflights short-circuit 204. opts: `origin` (`"*"` default, `"request"` echoes), `credentials`, `methods`, `headers`, `max_age`, `expose`. |
| `logger(log, opts?)` | one lib-log record per request after `next` returns — attrs `{ method, path, status, duration }`. opts: `emit` (default no-op — inject a lib-log backend arrow `(l, lvl, msg, attrs) ~> unit`), `now` (default constant 0 — inject a clock), `level`, `msg`. |
| `request_id(opts)` | propagate `x-request-id` or mint one (`opts.mint`, default rand-based — inject uuid-backed for production) into `ctx.state[opts.key]` and the response headers. |
| `session(store, opts?)` | cookie ↔ session record. `store` is a `{ get: (k) ~> v, set: (k, v) ~> u }` port record. opts: `cookie`, `mint`, `path`, `httponly`, `secure`. Helpers: `session_put(data)` (state fragment for handlers), `session_data(ctx)`, plus the `state.session = { id, data, is_new }` shape. |
| `auth(verify, opts?)` | `verify(ctx) ~> principal | null`; null → 401 (`www-authenticate` header) short-circuit; else principal merges into `ctx.state[opts.key]` (default `"principal"`, read via `principal(ctx)`). opts: `key`, `challenge`. |
| `csrf(check, opts?)` | `check(ctx) ~> Bool` on mutating methods (default POST/PUT/PATCH/DELETE); false → 403 short-circuit. opts: `methods`. |
| `body_limit(n)` | 413 short-circuit when the buffered body exceeds `n` bytes (`body_bytes` length, else `str.length(body)`). |

Helpers for middleware authors: `state_get(ctx, k)`,
`state_put(ctx, k, v)` (returns a new ctx).

## Layout

```text
src/index.hnl        the pipeline + v1 middleware (pure handl arrow dialect)
src/index.test.hnl   48 tests: chain ordering/short-circuit, ctx merge,
                     every middleware, session stub-store round-trip,
                     CORS preflight, auth 401, serve-shaped integration
pkg.hnl              package manifest — modules, license
ci.sh                bupha entry → scripts/hnl-gate.sh
scripts/hnl-gate.sh  typecheck + lint + test over .hnl sources
workflows/gate.yaml  generated gate include (handl-gates@0.3.2)
```

## Gate

The implementation uses the arrow dialect (`~>`), which the canonical
typecheck/lint tools skip — the gate reports it as dialect-skipped. The
real check is `hnl test` over `src/index.test.hnl` (needs
`HANDL_EXT_DIR` pointing at the extension `.so`s, as CI sets it):

```bash
HANDL_EXT_DIR=../.ci-target/dogfood/debug \
../.ci-target/dogfood/debug/hnl test src/index.test.hnl
```

or the full gate:

```bash
HNL=../.ci-target/dogfood/debug/hnl scripts/hnl-gate.sh
```

## License

MIT.

# Changelog — server-sequential-thinking (fork: thecryingman818181)

All notable changes to this fork are recorded here. Format loosely follows
"Keep a Changelog." The changes in 0.3.0 are applied to a **FORK for testing**;
the known-good production instance (`chic-caring`) is left **frozen as the
control** until the fork is verified.

Each fix below is tagged with a confidence level, because — per the working
discipline of this project — a fix is a *hypothesis to verify on the fork*,
not a guaranteed cure. Promote to production only after the fork passes the
test battery (see "How to verify").

---

## [0.3.0] - 2026-06-11 — "Stateful Hardening" (UNDER TEST on fork)

Three evidence-based fixes from a long forensic debugging session.

### Changed / Fixed

- **CORS: expose the `Mcp-Session-Id` response header.**
  Added `Access-Control-Expose-Headers: Mcp-Session-Id` to the CORS middleware.
  *Why:* a CORS-respecting client cannot *read* the session id off the response
  unless the server explicitly exposes it; without it, such a client has no id
  to reuse and is forced to open a fresh session every call. The server was
  omitting this standard header.
  *Confidence:* HIGH that this is a correct/standard fix. MEDIUM that it
  resolves a given client's per-call churn (some connectors churn by design,
  independent of this header).

- **Per-session MCP `Server` instance (was: one shared `Server`).**
  Each session now creates its **own** `Server` bound to its own transport and
  its own `SequentialThinkingServer`, instead of a single shared `Server`
  connected to every transport.
  *Why:* the single shared `Server` re-binds its active transport on each new
  connection, so when sessions interleave a reply can be routed to the wrong
  transport and the original request hangs with no response (observed as
  intermittent HTTP 499 after ~30s-4min, clearing on restart and recurring
  under load). Per-session isolation removes that race.
  *Confidence:* HIGH — the hang's behavior (intermittent, restart-clears,
  scales with concurrent sessions) matches this mechanism.

- **Return HTTP 404 (not 400) for an unknown/expired session.**
  A request carrying a session id the server no longer holds now returns
  `404` ("session not found / re-initialize") instead of `400`. A request with
  *no* session id that is not an `initialize` still returns `400`.
  *Why:* per the MCP spec, clients treat `404` as "session terminated, start a
  new one" and auto-recover; a `400` leaves spec-compliant clients wedged
  (observed: a client stuck failing until the connector was manually re-added).
  *Confidence:* MEDIUM — spec-aligned, but NOT yet verified end-to-end that a
  given client self-recovers on the 404.

- Version bumped `0.2.0` -> `0.3.0`.

### Known issues (observed this session, intentionally NOT changed here)

- **Session leak.** Clients that open a session and neither send `DELETE` nor
  hold an SSE stream leak a session until the 60-minute idle cleanup. Observed
  60+ sessions accumulating. Faster/idle-aware reaping is a candidate for a
  future release; left unchanged pending discussion (shortening the timeout
  also reaps legitimate held sessions sooner — a trade-off to decide).

- **`thoughtsRecordedThisSession` is not a correctness metric.** It is a
  per-session counter (`thoughtHistory.length`). It correctly stays at `1` for
  any client that opens a new session per call — this is expected, per the
  tool's own guidance string — and it climbs (verified `1->2->3->4`) on a client
  that holds one session. No code change; flagged so it is not mistaken for a
  bug.

### Deliberately NOT changed (rejected during review)

- **`chainId` / application-level session key** — rejected.
- **Making the server stateless** — rejected; would break cross-call thought
  history, branching, and the running count.

### How to verify (on the fork, before promoting)

1. Hold-session test: one client, one session, calls `thoughtNumber` 1->2->3 ->
   `thoughtsRecordedThisSession` should climb 1->2->3, `branches` populates on a
   `branchFromThought`/`branchId` call.
2. Hang test: drive overlapping sessions; confirm no `499`/multi-minute hangs
   (per-session `Server` should eliminate them).
3. Recovery test: let a session expire, reuse its id -> expect `404`, and
   confirm a spec-compliant client re-initializes on its own.
4. Compare every result against the frozen `0.2.0` control.

---

## [0.2.0] - (prior) — baseline (FROZEN as control)

- Stateful sequential-thinking over Streamable HTTP. Confirmed working in this
  session: the thought loop (`thoughtNumber` / `nextThoughtNeeded`), state
  accumulation on a held session (verified `1->2->3->4`), branching, and the
  dynamic `totalThoughts` bump. Known faults: the intermittent 499 hang, the
  60-minute reap + `400`-wedge on held-session clients, and the session leak
  noted above. This instance is the control; do not modify it.

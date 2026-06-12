# Daytona runtime for `verifiers`

A [Daytona](https://daytona.io) execution runtime for the v1 runtime layer of
[PrimeIntellect-ai/verifiers](https://github.com/PrimeIntellect-ai/verifiers). It fills the
`DaytonaRuntime` stub introduced in
[#1559](https://github.com/PrimeIntellect-ai/verifiers/pull/1559) and follows the structure
of the Modal runtime merged in
[#1594](https://github.com/PrimeIntellect-ai/verifiers/pull/1594).

## Contents

- `daytona.py` — the runtime (`DaytonaConfig` + `DaytonaRuntime`), drops in at
  `verifiers/v1/runtimes/daytona.py`
- `feat-add-daytona-runtime.patch` — the full change as a `git am`-able commit: the module,
  the `RuntimeConfig` union and `make_runtime` wiring, and the `daytona` dependency
  (`pyproject.toml` / `uv.lock`)

## Design notes

- **`run()`** — Daytona's exec returns a single combined output stream, so the runtime
  recovers the contract's stdout/stderr split in-band: stderr is redirected to a file during
  the run, emitted after a unique marker, and partitioned locally, with the exit code
  preserved. One round-trip, no extra API calls.
- **`public_url()`** — native Daytona preview links, with no tunnel process. On a private
  sandbox (the default) it mints a *signed* URL — which is what keeps the colocated
  user-simulator path (driven from the host via `public_url`) working without making ports
  world-readable. Signed for the 24h signing max unconditionally: the runtime's `timeout` is
  an inactivity backstop, not an absolute lifetime, so a shorter signature could go stale
  mid-rollout on an active sandbox; the link stops resolving when the sandbox is deleted
  anyway. A `public` sandbox returns the plain URL. `expose()` reaches host ports through
  `prime_tunnel`, same as the Modal and Prime runtimes.
- **Lifetime backstop** — Daytona has no absolute-lifetime knob, so `timeout` maps to
  inactivity auto-stop plus delete-on-stop (`auto_delete_interval=0`, which is also what
  the SDK's `ephemeral` flag aliases to): a leaked sandbox still removes itself.
- **Rate limiting** — sandbox creation is paced host-wide via the shared
  `creation_limiter` (`creates_per_sec`, default 5/s — the self-serve floor of 300/min);
  Daytona's creation limits are org-specific (300–600/min on self-serve tiers, custom on
  dedicated plans), so raise the knob to match your org or disable it. Tunnel starts share
  the runtimes' global `prime_tunnel` limiter. The SDK client's HTTP connection pool is
  also configurable (`connection_pool_maxsize`, default 250 — the SDK default) for runs
  driving many parallel sandboxes from one process.
- **Resources** — the Modal-units convention shared by all v1 runtimes, mapped to whole
  units; GPU specs split via `parse_gpu` (Daytona GPU sandboxes are ephemeral-only, set
  automatically).

## Applying

```sh
git clone https://github.com/PrimeIntellect-ai/verifiers
cd verifiers
git checkout feat/nano-as-v1
git am -3 path/to/feat-add-daytona-runtime.patch
```

The patch is based on the `feat/nano-as-v1` line as of June 12, 2026; `-3` resolves
`uv.lock` drift, or re-run `uv lock` after applying. Authenticate with
`DAYTONA_API_KEY` ([app.daytona.io](https://app.daytona.io)) and select the runtime with
`type = "daytona"`.

## Verification

Exercised live against Daytona's `us` region across the full contract: provisioning,
exec (split streams, exit codes, env, workdir), file IO, background processes, PEP 723
`uv` scripts resolving dependencies in-sandbox, signed `public_url` on a private sandbox
(content fetched from the host with no auth header; the unsigned URL correctly rejected
without a token), and teardown with server-side deletion confirmed — 17/17, re-run in full
after each revision. Exec round-trips in ~0.2s; provisioning takes ~3s on a cached image
(~13s on a cold registry pull); a signed preview URL is minted in ~0.1s with no tunnel
process.

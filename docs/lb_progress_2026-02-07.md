# Load-Balanced MongoDB Progress (2026-02-07)

## Goal

Enable Elixir `mongodb_driver` usage against servers that require `loadBalancer=true`, with practical priority on connect + query, and spec-aligned hardening in phases.

## Canonical Working URI Profile

```text
mongodb://app_user:***@home.brng.us:27018/app_user?authMechanism=PLAIN&authSource=%24external&tls=true&tlsAllowInvalidCertificates=true&retryWrites=false&loadBalancer=true
```

## What Was Implemented

### 1) Repro harness and baseline tests

- Created `lb_repro/` Mix project with integration-oriented LB tests.
- Added baseline connection helper and URI helper.
- Added integration tests that:
  - verify baseline connect result
  - run a real query path (`find_one`)
  - optionally run strict command-event assertions (`RUN_STRICT_LB_TESTS=true`)

### 2) LB URI + handshake behavior

- Added URL parsing support for:
  - `loadBalancer=true`
  - `tlsAllowInvalidCertificates=true`
- Enforced LB handshake expectations:
  - load-balanced mode flag set in hello path
  - required `serviceId` presence check during LB handshake
- Forced LB mode behavior for handshake protocol usage.

### 3) LB topology path

- Added load-balanced topology/server-type handling.
- Added no-monitor LB path and LB pool startup path.
- Added LB topology tests.

### 4) Connect/query reliability fixes for this environment

- Identified TLS requirement for the target endpoint.
- Added parser mapping so `tlsAllowInvalidCertificates=true` sets `ssl_opts: [verify: :verify_none]`.

### 5) Password-safe/auth lifecycle stabilization (fail-fast style)

- Removed race by making password storage synchronous.
- Reworked URL parser password handling to store a sealed password token rather than relying on a fragile linked process PID.
- Kept fail-fast behavior on invalid auth/password-safe state.

## Why These Changes Were Made

- To satisfy immediate practical requirement: connect and run queries with `loadBalancer=true`.
- To align with Mongo load-balancer spec incrementally without hiding errors.
- To eliminate observed reconnect/auth instability caused by `pw_safe` lifecycle/race issues.

## Validation Status

Validated against the canonical URI profile above:

- `MIX_ENV=test mix test` passes.
- `MIX_ENV=test MONGODB_URI_LB='...' mix test --only integration` passes.
- `MIX_ENV=test RUN_STRICT_LB_TESTS=true MONGODB_URI_LB='...' mix test --only integration` passes.
- Direct probe: `Mongo.ping` returns `{:ok, %{"ok" => 1.0}}`.

## Remaining Work

Still pending for full spec-compliance closure:

- serviceId-aware pool-clearing/event parity in all required paths
- cursor pinning behavior
- transaction pinning/unpinning behavior

These remain tracked under the LB implementation epic/work items in `bd`.

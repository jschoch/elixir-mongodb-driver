# MongoDB Load-Balancer Spec Gap Analysis (`mongodb_driver` 1.6.1)

Spec source: `docs/specs/mongodb-load-balancers-spec.md`

## Baseline

- Integration baseline run against `mongodb://home.brng.us:27018/test?loadBalancer=true` succeeded:
  - `baseline_lb_connect_result: {:ok, pid}`
- This confirms basic connect works for this endpoint, but does not confirm full spec compliance.

## Spec-to-Code Mapping

### 1) URI/config validation (`MongoClient Configuration`)

Spec requirements (section `MongoClient Configuration`):
- Driver must support `loadBalanced=true` URI option.
- Must reject invalid combinations (for example with disallowed topology options).
- Must support/validate in SRV TXT records.

Current code:
- `lb_repro/deps/mongodb_driver/lib/mongo/url_parser.ex` has no `loadBalanced` option in `@mongo_options`.
- `type` options do not include `load_balanced`.

Gap:
- No explicit parse, validation, or invalid-combination enforcement for `loadBalanced=true`.

### 2) SDAM topology behavior (`Server Discovery Logging and Monitoring`)

Spec requirements:
- Topology type must be `LoadBalanced` and remain so.
- Topology must contain one `LoadBalancer` server description.
- Driver must not start monitoring connection in LB mode.

Current code:
- `lb_repro/deps/mongodb_driver/lib/mongo/topology_description.ex` topology types:
  `:unknown | :single | :replica_set_no_primary | :replica_set_with_primary | :sharded`
- `lb_repro/deps/mongodb_driver/lib/mongo/topology.ex` always starts monitors in `update_monitor/1`.

Gap:
- No `LoadBalanced` topology type/server type.
- No LB-mode branch to disable monitors and force single-server selection semantics.

### 3) Handshake/serviceId (`Connection Pooling`)

Spec requirements:
- Handshake hello must include `loadBalanced: true`.
- In LB mode, hello response must include `serviceId`; missing `serviceId` must error.
- OP_MSG must be used for handshake in LB mode.

Current code:
- `lb_repro/deps/mongodb_driver/lib/mongo/mongo_db_connection.ex` handshake command does not set `loadBalanced: true`.
- No explicit `serviceId` validation path.

Gap:
- Missing LB-specific handshake fields and validation.

### 4) Session/cursor pinning (`Connection Pooling`, `Driver Sessions`)

Spec requirements:
- In LB mode, cursor and transaction flows must pin to a single connection as specified.
- Must handle unpin/release rules and error paths correctly.

Current code:
- `lb_repro/deps/mongodb_driver/lib/mongo/session.ex` has transaction/session state but no explicit LB pin/unpin model.
- No `serviceId`-aware pinning behavior found.

Gap:
- No explicit LB pinning implementation for cursor/transaction semantics.

### 5) Pool clearing + error handling (`Error Handling`, `Events`)

Spec requirements:
- Post-handshake errors must use `serviceId` to scope pool clearing.
- LB mode must not mark load balancer server unknown.
- Events should include `serviceId` where required.

Current code:
- No `serviceId` references in driver code.
- Existing topology error handling includes unknown-marking flows in `Mongo.Topology`.

Gap:
- Missing `serviceId`-scoped clearing and LB-specific SDAM/event behavior.

## Implementation Plan (for `elixir_mongo-2wx.3`)

1. Add `loadBalanced` parse + validation in URL parser and client option normalization.
2. Introduce load-balanced topology/server types and LB-specific server selection behavior.
3. Add LB handshake mode (`loadBalanced: true`) and `serviceId` enforcement.
4. Implement connection pinning state for cursor and transaction flows.
5. Implement `serviceId`-aware pool-clearing/error handling and event fields.
6. Add compliance-focused tests for each MUST/MUST NOT requirement above.

## Risks

- A full compliance implementation likely requires coordinated changes across SDAM, pooling, session, and event modules.
- If upstream driver already has partial/alternate LB support in unreleased code, pinning behavior may diverge from assumptions in this repo snapshot.

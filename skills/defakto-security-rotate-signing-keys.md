---
name: defakto-rotate-signing-keys
description: >-
  Rotate a Defakto trust domain's signing key set safely — prepare, activate, taint the old
  set, then remove it — without breaking workloads holding credentials chained to the outgoing
  key. Use for planned rotation, for an emergency rotation after suspected key compromise, or
  when investigating why workloads are rejecting each other's SVIDs after a rotation.
api: defakto-security-management-api
generated: '2026-08-12'
method: generated
source: grpc/defakto-security-trustdomainapi.proto, https://d.defakto.security/mint/operations/runbook-signing-key-rotation.md
operations:
  - TrustDomainSigningAuthorityStatus
  - ListTrustDomainKeys
  - CreateTrustDomainKey
  - EnableTrustDomainKey
  - DisableTrustDomainKey
  - DeleteTrustDomainKey
  - PrepareDeploymentKeySet
  - ActivateDeploymentKeySet
  - TaintDeploymentKeySet
  - RemoveDeploymentKeySet
  - ListTrustDomainDeployments
  - TrustDomainInfo
---

# Rotate a Defakto signing key set

Rotation is a **four-phase state machine**, not a single call. Each phase is its own RPC, and
skipping one breaks trust for every workload holding a credential chained to the outgoing key.

**Role required:** Manager, Administrator or Owner. Auditor and Operator can list keys and read
signing authority status but cannot mutate them.

## Phase 1 — Prepare

`PrepareDeploymentKeySet`

Creates the new key set and begins publishing its public material into the trust bundle.
Workloads start *trusting* the new key before anything is *signed* with it. Do not proceed until
`TrustDomainSigningAuthorityStatus` and `TrustDomainInfo` confirm the bundle has propagated —
`TrustDomainInfo` returns the bundle rotation schedule, which tells you the propagation window
to wait out.

Activating before the bundle reaches every agent is the single most common way to break a
rotation: workloads will be presented certificates signed by a key they have never seen.

## Phase 2 — Activate

`ActivateDeploymentKeySet`

Switches issuance to the new key set. From this point new SVIDs chain to the new key. Existing
SVIDs remain valid until they expire — check your configured SVID TTLs and the JWT-SVID policy
minimum TTL (5 minutes, enforced since spirl-server 0.36.0) to know how long the overlap runs.

## Phase 3 — Taint

`TaintDeploymentKeySet`

Marks the old key set as distrusted **while keeping it published**. This is the phase engineers
skip, and it is the one that matters. Defakto models taint as an OCSF *Update* rather than a
*Deactivate* precisely because a tainted key set stays in service: its IDs continue to be
distributed so workloads can see it is distrusted and rotate away from it. Removing instead of
tainting yanks the key out from under anything still holding a credential signed by it.

Taint is reversible — untaint is also modelled as an Update, and the event's `message` field is
what disambiguates the two in the audit stream.

## Phase 4 — Remove

`RemoveDeploymentKeySet`

Only after every credential signed by the old key set has expired. Confirm with
`SVIDIssuedEvents` (statistics service) that no recent issuance references the old `key_set_id`.

## Individual trust domain keys

Separate from deployment key sets, individual keys have their own lifecycle:
`ListTrustDomainKeys`, `CreateTrustDomainKey`, `EnableTrustDomainKey`, `DisableTrustDomainKey`,
`DeleteTrustDomainKey`. Same principle applies — disable before delete, never delete a key that
is still chained to live credentials.

## Verify from the audit stream

If OCSF audit logging is enabled (`ocsf.enabled: true` — it is **off by default**), every phase
emits an Entity Management event, `class_uid` 3004:

| Phase | OCSF activity | `activity_id` |
|---|---|---|
| Prepared | Create | 1 |
| Activated | Activate | 10 |
| Tainted | Update | 3 |
| Untainted | Update | 3 |
| Removed | Delete | 4 |

Issuance itself emits Authentication events, `class_uid` 3002, carrying `key_set_id` in the
event tags — that is how you confirm the cutover actually took.

## Cautions

- **No idempotency.** None of these RPCs takes a client token. A retried `PrepareDeploymentKeySet`
  after a timeout may create a second key set. Always `ListTrustDomainKeys` /
  `TrustDomainSigningAuthorityStatus` to read actual state before retrying — never retry blind.
- **Emergency rotation.** `spirlctl` 0.32.0 added the ability to force an early rotation from the
  CLI. It still runs the same four phases; it does not let you skip taint.
- **Key material location.** If the trust domain uses an external Key Manager (AWS KMS, Azure Key
  Vault, GCP Cloud KMS) or key wrapping, removal timing interacts with that provider's
  soft-delete. GCP KMS in particular honours `destroyScheduledDuration` (24 hours to 120 days,
  spirl-server 0.38.0+), so a "removed" key may remain recoverable well past the API call.

Full runbook: <https://d.defakto.security/mint/operations/runbook-signing-key-rotation.md>

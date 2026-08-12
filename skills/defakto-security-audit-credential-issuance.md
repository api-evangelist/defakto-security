---
name: defakto-audit-credential-issuance
description: >-
  Answer "which non-human identities exist, what credentials were minted for them, by which
  agent, and what failed" using the Defakto statistics and workloads services plus the OCSF
  audit stream. Use for NHI inventory, incident investigation after a suspected credential
  compromise, or explaining why a workload is not receiving an SVID.
api: defakto-security-management-api
generated: '2026-08-12'
method: generated
source: grpc/defakto-security-statisticsapi.proto, grpc/defakto-security-workloadsapi.proto, https://d.defakto.security/mint/configuration/telemetry/spirl-telemetry-audit-logging.md
operations:
  - ListTrustDomainWorkloads
  - SVIDIssuedEvents
  - HealthEvents
  - ListActivityFeed
  - ListAgents
  - DescribeAgents
  - ListServers
  - Query
  - Count
  - ListWorkloads
  - ListNodes
---

# Audit credential issuance in Defakto

Everything below is **Auditor-role readable** — the lowest role in the model. You do not need
elevated access to run an NHI inventory or an incident timeline.

## 1. Inventory the workloads

`ListTrustDomainWorkloads` is the NHI census. Required: `trust_domain_id` and `page_size` (1–1000).

- Paginate with `page_token`; an **empty** `next_page_token` means the final page.
- `issued_from` / `issued_to` bound the `credentials_issued` count. **Defaults to the last 30
  days** — if you are reconstructing an older incident you must set these explicitly or you will
  read zeros and conclude nothing happened.
- `breakdowns` accepts `BREAKDOWN_DIMENSION_SVID_TYPE` and `BREAKDOWN_DIMENSION_ISSUER_TYPE`.
  Empty means totals per `spiffe_id` only.
- `include_summary: true` adds totals across **all** matching workloads, not just the current page.
  Use it — otherwise you have to sum pages yourself and will get it wrong on a filtered query.
- `query_filters` on this service supports `spiffe_id` (`EQUAL`, `PREFIX`) and `issuer_parent_id`
  (`EQUAL`), combined with AND. `PREFIX` on `spiffe_id` is how you scope to a realm or a path
  segment.

## 2. Pull the issuance record

`SVIDIssuedEvents` returns the per-credential audit spine. Each `SVIDIssuedEvent` carries the
`spiffe_id`, the issuing `agent_id`, the `issuer_id` and `issuer_parent_id` chain, the
`key_set_id`, and typed `JWTAttributes` / `X509Attributes` (including `authority_key_id` and
`subject_key_id`).

This is the join that answers a compromise question: given a leaked certificate, the
`subject_key_id` and `key_set_id` identify which key set signed it and which agent requested it.

`SVIDIssuedEventFilter` supports filtering by `spiffe_id` among others.

## 3. Reconstruct the timeline

`ListActivityFeed` is the unified feed. Entries are polymorphic — inspect which attribute block
is populated:

| Attributes | Meaning |
|---|---|
| `WorkloadCredentialIssuedAttributes` | A credential was minted |
| `WorkloadCredentialErrorAttributes` | Issuance failed — carries `issuer_id`, `spiffe_id`, `agent_id` |
| `AgentAttestationAttributes` | An agent attested — carries `cluster_id`, `cluster_version_id` |
| `AuditLogAttributes` | Control-plane change — carries `request_id` |
| `FederationLinkPolledAttributes` | A federation link was polled |

`AuditLogAttributes.request_id` is the correlation key. There is **no client-supplied
correlation header** on the request path — tracing is server-emitted and read back here.

## 4. Check the infrastructure side

- `ListAgents` / `DescribeAgents` — agents by `cluster_id` and `realm_id`. A workload that is not
  getting an SVID usually has no healthy agent, not a policy problem.
- `ListServers` — Trust Domain Server instances with `ServerConfigStatus`.
- `HealthEvents` — filterable by `by_source_id` and `by_trust_domain_id`.
- `Query` / `Count` — generic time-series over `Series`, `Float64Value` and `CountValue`.

## 5. The OCSF stream is the other half

The API is a **pull** surface. The **push** surface is the OCSF 1.8.0 audit stream, and the two
carry overlapping but non-identical detail — neither is a superset.

- **Off by default.** `ocsf.enabled: false` unless someone turned it on. If an investigation
  needs it and it was never enabled, that history does not exist.
- NDJSON on stdout. The server does not ship events anywhere; routing to a SIEM is the operator's job.
- `class_uid` **3002** Authentication = one per SVID mint attempt, success **and** failure.
  `class_uid` **3004** Entity Management = signing key set lifecycle.
- `severity_id` 1 on success / 3 on failure; `status_id` 1 success / 2 failure.
  `status_code` carries the **gRPC status code** on failure, or `Unchanged` on an idempotent
  success that changed nothing.
- Origin tags on every event: `trust_domain_id` (`td-…`), `trust_domain_deployment_id` (`tdd-…`);
  Authentication events add `cluster_id` (`c-…`), `cluster_version_id` (`cv-…`) and `realm`.

**The detail that matters for incident work:** when issuance fails *before* the workload identity
resolves — a missing argument, or attributes matching no registered identity — the event is still
emitted, attributed to the canonical **Unknown** subject (`user.name: "unknown"`,
`user.type: "Unknown"`). Unauthorized attempts are audited, not dropped. Search for Unknown
subjects to find workloads probing for identities they are not entitled to.

## Diagnosing "my workload gets no SVID"

Work outward in this order:

1. `ListAgents` — is there a healthy agent on that node/cluster?
2. `AgentAttestationAttributes` in `ListActivityFeed` — did the agent attest at all?
3. `WorkloadCredentialErrorAttributes` — did issuance fail, and against which `spiffe_id`?
4. OCSF Authentication events with Unknown subject — did workload attestation match no identity?
5. `TrustDomainSigningAuthorityStatus` — is there an active key set to sign with?

On Kubernetes, also confirm the pod carries the `k8s.spirl.com/spiffe-csi: enabled` label so the
admission controller injects the Workload API socket. Without it the workload has nothing to call.

Runbooks: <https://d.defakto.security/mint/operations/runbook-agent.md>,
<https://d.defakto.security/mint/operations/runbook-server.md>

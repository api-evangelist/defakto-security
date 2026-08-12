---
name: defakto-onboard-trust-domain
description: >-
  Stand up a Defakto SPIFFE trust domain and attach a Kubernetes cluster to it so workloads
  begin receiving X.509-SVIDs and JWT-SVIDs automatically. Use when onboarding a new
  environment onto Defakto, or when a workload reports it cannot fetch an SVID because its
  cluster is not registered.
api: defakto-security-management-api
generated: '2026-08-12'
method: generated
source: grpc/defakto-security-trustdomainapi.proto, grpc/defakto-security-clusterapi.proto, https://d.defakto.security/mint/quick-start.md
operations:
  - CreateTrustDomain
  - RegisterTrustDomain
  - ListTrustDomains
  - TrustDomainInfo
  - ListTrustDomainDeployments
  - CreateCluster
  - ListClusters
  - DescribeClusters
  - NewClusterVersion
  - ActivateClusterVersion
  - ListClusterVersions
---

# Onboard a Defakto trust domain and cluster

## Before you start

- The API is **gRPC only** on `https://api.defakto.security:443`. There is no REST surface —
  `api.defakto.security/openapi.json` answers `415 application/grpc`. Drive it with the Go SDK
  (`github.com/spirl/spirl-sdk-go`), the `spirlctl` CLI, or the OpenTofu provider.
- Authenticate first. Interactive work uses `spirlctl login`; automation uses a service account
  Ed25519 key (`--service-account-key-id sak-… --private-key-file …`) or Workload Identity
  Federation. See `authentication/defakto-security-authentication.yml`.
- **Role required:** creating, registering or updating a trust domain needs **Administrator**
  or **Owner**. Creating a cluster needs **Operator** or above, or **Realm Admin** within the
  target realm. A caller with a lower role gets gRPC `PERMISSION_DENIED` — which the Go SDK
  does *not* map to a typed error, so check `status.Code(err)` directly.

## Steps

1. **Check what already exists.** Call `ListTrustDomains` before creating anything. There is no
   idempotency key on `CreateTrustDomain`, so a blind retry after an ambiguous failure can
   create a duplicate trust domain. Listing first is the only protection available.

2. **Create or register the trust domain.**
   - `CreateTrustDomain` for a Defakto-managed trust domain.
   - `RegisterTrustDomain` for a self-hosted one. As of `spirlctl` 0.28.0 new trust domains are
     treated as self-hosted unless `--self-hosted=false` is passed.
   - Optionally set the JWT issuer for the `iss` claim on JWT-SVIDs: built-in (computed
     default), disabled (no `iss` claim), or a custom HTTPS OIDC issuer.

3. **Confirm signing authority is live.** Call `TrustDomainInfo` and
   `TrustDomainSigningAuthorityStatus`. `TrustDomainInfo` returns the bundle rotation schedule
   and signing key metadata. Do not attach clusters until the signing authority reports healthy —
   a cluster attached to a trust domain with no active key set will attest agents but fail
   issuance, and the failure surfaces as an OCSF Authentication event with `status_id: 2`
   rather than as an obvious control-plane error.

4. **Register a deployment.** `ListTrustDomainDeployments` to see what exists. When creating one
   via the CLI, pass `--output-helm-values ./values.yaml` to get pre-formatted Helm config for
   deploying Trust Domain Servers into it.

5. **Create the cluster.** `CreateCluster` against the trust domain. If the organization uses
   realms, set the realm so administration can be delegated — a cluster's realm also becomes a
   SPIFFE ID path prefix, so this is a naming decision, not just an access-control one, and it
   cannot be changed after workloads have identities minted under it.

6. **Create and activate a cluster version.** `NewClusterVersion` then `ActivateClusterVersion`.
   The version carries the agent attestation config. Prefer identifying the cluster by **name**
   via `agent.auth.clusterName` (spirl-server 0.37.0+) rather than passing a Defakto-generated
   `c-…` cluster ID between provisioning and deployment pipelines.

7. **Verify.** `DescribeClusters` for detailed state, `ListClusterVersions` to confirm the right
   version is active, then `ListTrustDomainWorkloads` (workloads service) to see SPIFFE IDs
   appearing. `ListWorkloads` and `ListNodes` on the cluster service give the per-cluster view.

## Conventions that will bite you

- **`ListClusters` vs `ListClustersV2`.** Both exist on the current contract. `ListClusters`
  historically omitted Linux-runtime clusters ("node-groups"); that was fixed in `spirlctl`
  0.30.0. Prefer `ListClustersV2` and verify node-groups appear.
- **Pagination is not universal.** Unbounded reads (workloads, statistics, activity feed) use
  `page_size` (1–1000) plus `page_token`, where an **empty** `next_page_token` means the final
  page. Bounded control-plane lists return everything at once with no token.
- **`include_dynamic_data`.** `ListRealms` returns database rows only unless you set this, which
  joins live event-derived cluster/agent/workload statistics. Counts will look wrong if you omit it.
- **Teardown order.** A trust domain cannot be deleted while clusters are still registered to it.

## Error handling

The envelope is `google.rpc.Status`, never RFC 9457 problem+json. The Go SDK types only
`NOT_FOUND` → `ErrNotFound` and `UNAUTHENTICATED` → `ErrUnauthenticated`. Everything else —
including `PERMISSION_DENIED`, `ALREADY_EXISTS`, `FAILED_PRECONDITION` and `RESOURCE_EXHAUSTED` —
falls through untyped. See `errors/defakto-security-problem-types.yml`.

An `ErrOutOfDate` means the server returned a field the installed SDK does not recognise:
upgrade `github.com/spirl/spirl-sdk-go`, do not ignore it.

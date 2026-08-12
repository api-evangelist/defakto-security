---
name: defakto-secretless-cicd
description: >-
  Give a CI/CD pipeline a Defakto identity with no stored secret — either by federating the
  pipeline's own OIDC token into a Defakto service account (Workload Identity Federation), or
  by issuing SPIFFE identities to the pipeline's jobs through a CI/CD profile. Use when
  removing long-lived API keys from GitHub Actions, GitLab, Jenkins or Terraform Cloud.
api: defakto-security-management-api
generated: '2026-08-12'
method: generated
source: grpc/defakto-security-accessapi.proto, grpc/defakto-security-cicdapi.proto, https://d.defakto.security/iam/service-accounts.md
operations:
  - CreateServiceAccount
  - ListServiceAccounts
  - GetServiceAccountInfo
  - UpdateServiceAccountRole
  - CreateServiceAccountKey
  - UpdateServiceAccountKeyStatus
  - DeleteServiceAccountKey
  - DeleteServiceAccount
  - ListRoles
  - CreateCICDProfile
  - ListCICDProfiles
  - LinkCICDProfile
  - UnlinkCICDProfile
  - ListCICDProfileLinks
  - DeleteCICDProfile
---

# Secretless CI/CD with Defakto

There are two distinct things people mean by "CI/CD identity" in Defakto, and they use different
services. Pick the right one first.

| Goal | Surface | Service |
|---|---|---|
| The pipeline needs to **call the Defakto API** (run `spirlctl`, apply the OpenTofu provider) | Service account + WIF configuration | `accessapi` |
| The pipeline's **jobs need SPIFFE identities** to talk to your own systems | CI/CD profile | `cicdapi` |

They are often both wanted. Do the service account first — you need API access to configure
anything else.

---

## Path A — Service account with Workload Identity Federation (no stored key)

1. **Register the WIF issuer.** An org-wide record holding the external OIDC provider and its
   key material. **Administrator or Owner only.** One issuer per external IdP (GitHub Actions,
   GitLab, Jenkins, Terraform Cloud). CLI: `spirlctl iam wif-issuer` (0.35.0+).

2. **Pick the role, then create the service account.** `ListRoles`, then `CreateServiceAccount`.
   Three hard constraints from the roles model:
   - A service account **cannot hold Owner**.
   - You can only create a service account with a role **equal to or lower than your own**.
   - Service accounts **cannot create service accounts**, so a pipeline cannot bootstrap another.

   Grant the least role that works. A pipeline that only reads should be **Auditor**; one that
   manages clusters needs **Operator**; only key-set rotation needs **Manager**.

3. **Attach the WIF configuration.** Bind the service account to the issuer and declare which
   JWT claims must match — repository, ref, environment, workspace, whatever the IdP asserts.
   The claim filter is the entire security boundary here: a WIF config that matches only on
   issuer will accept **any** job from that platform. CLI:
   `spirlctl iam service-account wif-config` (0.35.0+).

4. **Use it.** The pipeline presents its short-lived platform-issued OIDC token and receives a
   normal service account session carrying that account's role and realm assignments. Nothing is
   stored. For OpenTofu/Terraform this is documented as secretless Terraform authentication.

5. **Audit it.** WIF sessions are recorded with the OIDC issuer, the matched claims and the
   session ID, and every subsequent call in that session carries the same session ID — so all
   activity traces back to the originating token exchange. Read it via `ListActivityFeed`.

## Path A′ — Ed25519 key, when WIF is not available

Only when the platform issues no OIDC token.

- `CreateServiceAccountKey` returns an Ed25519 key pair. The **private key never leaves you** —
  it signs an authentication challenge; it is not transmitted.
- Login: `spirlctl login --service-account-key-id sak-… --private-key-file …`.
  OpenTofu: `sa_key_id` + `sa_private_key = file(...)`.
- **Rotate with two keys, never one.** Multiple keys per account exist specifically for
  zero-downtime rotation: `CreateServiceAccountKey` → deploy → verify →
  `UpdateServiceAccountKeyStatus` to disable the old one → `DeleteServiceAccountKey`.
  Disable before delete, so a rollback is possible.
- Note the irony and act on it: this path stores a long-lived private key, which is the exact
  thing Defakto exists to eliminate. Treat it as a migration step toward Path A.

---

## Path B — SPIFFE identities for pipeline jobs

1. `CreateCICDProfile` — maps the CI/CD system's identity to SPIFFE ID issuance rules.
2. `LinkCICDProfile` — link the profile to a cluster and trust domain. A cluster carries at most
   one `ci_cd_profile_id`, so linking a second profile to the same cluster replaces the binding
   rather than adding to it.
3. `ListCICDProfileLinks` to verify; `UnlinkCICDProfile` / `DeleteCICDProfile` to tear down.

Platform quick starts: GitHub Actions, GitLab self-hosted runners, and Jenkins are each
documented separately under `/mint/integration/ci-cd/`.

**Role required:** create/delete a CI/CD profile and link/unlink needs **Operator** or above.

---

## Cautions

- **No idempotency anywhere.** `CreateServiceAccount`, `CreateServiceAccountKey` and
  `CreateCICDProfile` take no client token. A retry after a timeout creates a duplicate — and a
  duplicate service account key is a duplicate credential. Always `ListServiceAccounts` /
  `GetServiceAccountInfo` before retrying.
- **Ownership rules.** The creating user is the owner. Non-admin roles can update or delete only
  the service accounts they created, so an offboarded employee's service accounts become
  admin-only to manage.
- **`PERMISSION_DENIED` is untyped.** The role model makes permission failures the most likely
  error on this path, yet the Go SDK leaves `PERMISSION_DENIED` unmapped. Inspect
  `status.Code(err)` from `google.golang.org/grpc/status` directly.
- **Least privilege is enforced at creation, not continuously.** `UpdateServiceAccountRole` can
  raise a role later; audit `ListRoleAssignments` periodically.

Docs: <https://d.defakto.security/iam/service-accounts.md>,
<https://d.defakto.security/iam/wif-issuers.md>,
<https://d.defakto.security/iam/terraform-wif.md>

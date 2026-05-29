---
name: orbit-design
description: Design spec for orbit — a Rust CLI for secret provisioning, rotation, and revocation
metadata:
  type: project
---

# Orbit Design Spec

**orbit** is a Rust CLI for secret provisioning, rotation, and revocation. It uses a declarative config model (`orbit.json`) and a trait-based adapter system that makes adding new secret types trivial.

**Why:** Grafana's oncall and federal teams have experienced multiple security incidents requiring manual secret rotation across 5+ environments and 8+ secret types. An existing runbook and partial scripts exist, but no unified tool ties them together for incident response.

**Why not Terraform:** Rotation is a procedure (create → store → revoke), not desired state. Terraform is appropriate for provisioning Vault paths and ExternalSecrets CRDs — not for "rotate this right now."

---

## Commands

```
orbit generate                                           # interactive wizard → writes orbit.json
orbit list      [file|path]                              # list secrets, read-only, no side effects
orbit validate  [file|path]                              # schema check only, no side effects
orbit provision [file|path] [--secret <name>] [--dry-run]  # idempotent bootstrap
orbit rotate    [file|path] [--secret <name>] [--dry-run]  # create new + store + revoke old
orbit revoke    [file|path] [--secret <name>] [--value <val>] [--dry-run]  # revoke at source + remove from storage
orbit list-types                                         # show all registered adapter types
```

**Path resolution:** if given a directory, orbit recursively finds all `orbit.json` files and processes each. `--secret <name>` scopes to a single named secret within a file; ignored when processing a directory.

**`--dry-run`:** runs all read/check operations (`exists()`, `fetch()`) but never calls `create()`, `store()`, `revoke()`, or `delete()`. Prints what would change.

---

## Config Format

A single `orbit.json` can define one or many secrets. This allows related secrets (e.g., all secrets for a Tanka environment) to be co-located and managed as a batch while still being individually addressable via `--secret <name>`.

```json
{
  "secrets": [
    {
      "name": "loki-writer-token",
      "type": "grafana-access-policy-token",
      "url": "https://grafana.example.com/api/v1/",
      "options": {
        "policy_name": "loki-writer",
        "expiry_days": 90
      },
      "storage": {
        "type": "vault",
        "path": "loki/dev-us-gov-east-1/federal-loki/amcreds/token"
      }
    },
    {
      "name": "traces-access-token",
      "type": "grafana-access-policy-token",
      "url": "https://traces.example.com/admin/api",
      "auth": {
        "type": "vault",
        "path": "traces/dev-us-gov-east-1/admin/token",
        "field": "token"
      },
      "options": {
        "policy_name": "traces-writer",
        "expiry_days": 90
      },
      "storage": {
        "type": "vault",
        "path": "traces/dev-us-gov-east-1/amcreds/token"
      }
    }
  ]
}
```

**Auth resolution order** (per secret):

1. If an `auth` block is present — fetch credentials from the specified backend before calling the adapter. Uses ambient backend auth (e.g. `VAULT_TOKEN`) to perform the lookup.
2. Otherwise — resolve from environment variables (`GRAFANA_API_TOKEN`, `GRAFANA_COM_TOKEN`, etc.)
3. Otherwise — prompt interactively

**`auth` block — shorthand (single credential):**

```json
"auth": {
  "type": "vault",
  "path": "traces/dev/admin/token",
  "field": "token"
}
```

Equivalent to a single-entry `credentials` list with `key` defaulting to the `field` name.

**`auth` block — full form (multiple credentials):**

```json
"auth": {
  "type": "vault",
  "credentials": [
    { "key": "client_id",     "path": "app/oauth/creds", "field": "client_id" },
    { "key": "client_secret", "path": "app/oauth/creds", "field": "client_secret" }
  ]
}
```

Each entry in `credentials` fetches one value and names it. All resolved values are collected into an `authMap` (`HashMap<String, String>`) and passed to the adapter. The adapter extracts whichever keys it needs — orbit does not prescribe how the credentials are used.

**`credentials` entry fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `key` | yes | Name used to reference this credential in the adapter (`authMap` key) |
| `path` | yes | Path to the secret in the backend |
| `field` | no | Specific field within that secret. If omitted, the entire secret value is used. |

The adapter declares (via `options_schema()`) what keys it expects from `authMap`. The config does not need to specify how credentials are presented to the target API — that is the adapter's responsibility.

---

## Traits

Three traits, separated by operation so adapters can implement only what they support.

### `ProvisionSecret` — used by `provision`

```rust
trait ProvisionSecret {
    fn type_name() -> &'static str;

    /// Fields this adapter needs in "options" — drives the `generate` wizard
    fn options_schema() -> Vec<OptionField>;

    /// Check whether the secret exists in the source system
    fn exists(&self, config: &AdapterConfig, auth: &AuthContext) -> Result<bool>;

    /// Fetch the current secret value from the source system.
    /// Returns None if the API doesn't support value retrieval after creation.
    fn fetch_current(&self, config: &AdapterConfig, auth: &AuthContext) -> Result<Option<Secret>>;

    /// Create a new secret in the source system
    fn create(&self, config: &AdapterConfig, auth: &AuthContext) -> Result<Secret>;
}
```

### `RotateSecret: ProvisionSecret` — used by `rotate` and `revoke`

```rust
trait RotateSecret: ProvisionSecret {
    /// Revoke/invalidate a secret in the source system
    fn revoke(&self, secret: &Secret, config: &AdapterConfig, auth: &AuthContext) -> Result<()>;
}
```

An adapter that only supports creation (e.g., a write-once token with no revocation API) implements `ProvisionSecret` alone and works with `provision`. Full-rotation adapters implement `RotateSecret`.

### `StoreSecret` — used by all mutating commands

```rust
trait StoreSecret {
    fn type_name() -> &'static str;
    fn exists(&self, config: &StorageConfig, auth: &AuthContext) -> Result<bool>;
    fn store(&self, secret: &Secret, config: &StorageConfig, auth: &AuthContext) -> Result<()>;
    fn fetch(&self, config: &StorageConfig, auth: &AuthContext) -> Result<Option<Secret>>;
    fn delete(&self, config: &StorageConfig, auth: &AuthContext) -> Result<()>;
}
```

### `AuthContext` — resolved credentials map

```rust
struct AuthContext {
    /// Named credentials resolved from the auth block (or env/prompt fallback).
    /// Adapters extract the keys they need: auth.get("client_id"), auth.get("token"), etc.
    credentials: HashMap<String, String>,
}
```

### `OptionField` — for the `generate` wizard

```rust
struct OptionField {
    name: &'static str,
    description: &'static str,
    required: bool,
    kind: FieldKind,  // Text | Url | Integer | Bool | Select(Vec<&'static str>)
}
```

---

## Command Behaviors

### `orbit provision` — idempotent bootstrap

```
                    ┌─ exists in storage? ──YES──→ report "already provisioned", exit 0
config loaded ──────┤
                    └─ NO ──→ exists in source? ──YES──→ fetch_current* + store
                                                └─ NO ──→ create + store

* If fetch_current returns None: create new + store. No revoke — nothing existed before.
```

### `orbit rotate` — always creates new and revokes old

1. Load + validate config
2. Resolve auth: fetch from backend (`auth` block) → env var → prompt
3. `storage.fetch()` → retrieve current secret value (needed for revoke in step 6)
4. `adapter.create()` → create new secret
5. `storage.store()` → write new secret to backend
6. `adapter.revoke(old)` → invalidate old secret
   - If step 3 returned nothing (no secret in storage yet): warn "no existing secret found to revoke, proceeding with create + store only" and skip revoke. Suggest running `orbit provision` for first-time setup.
7. Report result

### `orbit revoke` — kill a secret without replacement

```
config loaded → storage.fetch()
                    │
                    ├─ found → adapter.revoke() → storage.delete() → report
                    │
                    └─ not found → warn "nothing stored"
                                   (--value flag allows direct input as fallback)
```

### `orbit generate` — wizard flow

1. Select secret type (from registered adapters via `list-types`)
2. Prompt for each field in `adapter.options_schema()`
3. Ask: does this API require a credential stored in a backend? If yes:
   - Select backend type (`vault`, `1password`)
   - Prompt for path and field name
4. Select storage backend
5. Prompt for storage path/key
6. Write `orbit.json`

### `orbit list` — read-only summary

```
FILE                              NAME                  TYPE                          STORAGE
environments/dev/orbit.json       loki-writer-token     grafana-access-policy-token  vault:loki/dev/.../token
environments/dev/orbit.json       mimir-writer-token    grafana-access-policy-token  vault:mimir/dev/.../token
```

### `--dry-run` output example

```
[dry-run] orbit rotate ./environments/

  environments/dev/orbit.json  loki-writer-token    grafana-access-policy-token  → ROTATE
  environments/dev/orbit.json  mimir-writer-token   grafana-access-policy-token  → ROTATE
  environments/prod/orbit.json slack-oncall-token   slack-token                  → ROTATE

3 secrets would be rotated. Run without --dry-run to apply.
```

---

## Project Structure

```
orbit/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── cli.rs                        # clap command definitions
│   ├── config.rs                     # orbit.json loading, validation, path walking
│   ├── adapters/
│   │   ├── mod.rs                    # ProvisionSecret + RotateSecret traits + registry
│   │   ├── grafana_access_policy.rs  # type: grafana-access-policy-token
│   │   ├── gcom_token.rs             # type: gcom-api-token
│   │   ├── aws_iam_key.rs            # type: aws-iam-access-key
│   │   ├── gcp_service_account.rs    # type: gcp-service-account-key
│   │   └── slack_token.rs            # type: slack-token
│   ├── store/
│   │   ├── mod.rs                    # StoreSecret trait + backend registry
│   │   ├── vault.rs                  # type: vault (KV v2)
│   │   └── onepassword.rs            # type: 1password
│   ├── commands/
│   │   ├── provision.rs
│   │   ├── rotate.rs
│   │   ├── revoke.rs
│   │   ├── generate.rs
│   │   ├── list.rs
│   │   └── validate.rs
│   └── auth/
│       └── mod.rs                    # resolution chain: auth block (backend fetch) → env var → prompt
├── examples/
│   ├── grafana-access-policy-token.orbit.json
│   ├── gcom-api-token.orbit.json
│   └── aws-iam-key.orbit.json
└── README.md
```

---

## v1 Adapters

| Type | Auth | Notes |
|------|------|-------|
| `grafana-access-policy-token` | `GRAFANA_API_TOKEN` env | Grafana Cloud API |
| `gcom-api-token` | `GRAFANA_COM_TOKEN` env | 3-step create/swap/delete |
| `aws-iam-access-key` | ambient AWS credentials | `irm/dev/scripts/aws_email_credentials/rotate.sh` maps this out |
| `gcp-service-account-key` | ambient gcloud auth | GCP SA key JSON |
| `slack-token` | prompted | 1Password → Vault |

## v1 Storage Backends

| Type | Auth |
|------|------|
| `vault` | `VAULT_TOKEN` or ambient vault login |
| `1password` | `op` CLI ambient / `OP_SERVICE_ACCOUNT_TOKEN` |

---

## Verification

- `cargo test` — unit tests per adapter (mock HTTP, assert secret shape)
- `orbit validate examples/` — all example configs pass schema check
- `orbit list examples/` — all adapters listed correctly
- `orbit provision --dry-run examples/grafana-access-policy-token.orbit.json` — prints expected state without calling APIs
- End-to-end: `orbit rotate` against dev Loki instance, verify new token written to Vault, old token rejected by API

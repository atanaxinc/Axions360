## Description

<!-- What does this PR do? What problem does it solve? Link the related Issue. -->

Closes #

## Type of change

- [ ] Bug fix (non-breaking)
- [ ] New feature (non-breaking)
- [ ] Breaking change (requires migration plan — see CONTRIBUTING.md)
- [ ] Documentation update
- [ ] Security patch

---

## Checklist

> Read `CONTRIBUTING.md` before submitting. PRs that fail this checklist will not be reviewed.

### Schema & Data
- [ ] Entity Contract V1 respected (`id`, `ulid`, `tenant_id`, `status_code` via Lookup Table, `row_version` if Tier 1/2, timestamps `WITH TIME ZONE`)
- [ ] New entity tables have an entry in `entity_registry` with the correct tier and protections
- [ ] No hardcoded ENUMs — Lookup Tables with i18n sidecar only
- [ ] Migrations are strictly additive (Law 1 — no DROP, no RENAME on active columns)
- [ ] No column removed or renamed without a deprecation cycle (Law 2)
- [ ] Monetary amounts stored as `BIGINT` minor units — never `DECIMAL` or `FLOAT`
- [ ] Public-facing entities use `ULID` — never integer `id` in URLs or API responses

### Architecture
- [ ] Tech-agnosticism rule respected — architectural rules declared in the database, not only in code
- [ ] No module accesses another module's tables directly
- [ ] Inter-module communication via Events (opscodes) only
- [ ] Financial operations create an entry in `transactions` with `operation_code` + `source_opscode_code`
- [ ] Sensitive actions logged in `audit_logs` with `opscode_code`
- [ ] `audit_logs` and `domain_events` are append-only — no UPDATE, no DELETE

### Versioning
- [ ] Sync Protocol unchanged, or a new version decoder has been added (Law 3)
- [ ] `module_activations` updated if module version number changes
- [ ] Compatibility with active `release/` branches verified

### i18n & Opscodes
- [ ] User-visible content has a `_translations` or `_i18n` table
- [ ] New business events have an opscode declared in the catalogue with FR and EN translations minimum
- [ ] New financial operations have an `operation_code` in the catalogue

### Security
- [ ] PII fields are encrypted with `blind_index` and `key_version_id`
- [ ] No cryptographic key stored in the database — `key_ref` (KMS) only
- [ ] Financial operations have an `idempotency_key` with a UNIQUE constraint
- [ ] No secrets hardcoded in code or configuration files

### Tests
- [ ] Unit tests on Services — coverage ≥ 90%
- [ ] Feature tests on API endpoints — coverage ≥ 80%
- [ ] Sync Engine non-regression tests passed (if applicable)
- [ ] Architecture tests passed (no inter-module coupling violations)

---

## Migration plan (if breaking change)

<!-- Required if this PR introduces a breaking change. Describe:
- Which release/ branches are affected
- How the field/route will be deprecated
- Timeline for removal
- How on-premise clients will be migrated -->

N/A

---

## Notes for reviewers

<!-- Anything specific reviewers should pay attention to -->

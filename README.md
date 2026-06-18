<div align="center">

# Axions360

### The Open-Core Business Operating System for Africa.

**Multi-Tenant · Multi-Language · Multi-Currency · Multi-Country · Multi-Accounting Framework · Offline-First · Activation-Driven**

From a single POS terminal to a multi-country enterprise group.
No migration. Just activation.

[Website](https://axions360.com) · [Documentation](https://docs.axions360.com) · [Community](https://github.com/atanaxinc/axions360/discussions) · [Issues](https://github.com/atanaxinc/axions360/issues)

![License](https://img.shields.io/badge/license-AGPL--3.0-blue)
![Services](https://img.shields.io/badge/services-Commercial-lightgrey)
![Status](https://img.shields.io/badge/status-early%20development-orange)
![PHP](https://img.shields.io/badge/PHP-8.3%2B-777BB4)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1)
![Built in Africa](https://img.shields.io/badge/built%20in-Africa%2C%20ready%20for%20the%20world-009900)

</div>

---

## Why Axions360?

Most business software forces companies to migrate when they grow.

Axions360 does not.

A restaurant owner starts with Synchro POS today. Three years later, when they open a hotel and a second location, they activate HR, Finance, and multi-site reporting — without exporting a single row of data. Their employees, transactions, documents, and history are already there.

**Start small. Activate more. Never migrate.**

---

## Core Capabilities

These are architectural guarantees built into every layer of the system — not features to be added later.

### Multi-Tenant by Design

One platform. Unlimited organizations. Complete data isolation enforced at every layer — application scope, query filters, and database Row Level Security.

### Multi-Language by Design

Languages are data, not files. Every label, status, document, notification, permission, and business term is stored in the database with its translations. No framework language files. No hardcoded strings.

Supports unlimited languages through database-driven translations.

### Multi-Currency by Design

Transactions, accounting, and reporting support multiple currencies natively. The financial engine distinguishes between:

- **Transaction currency** — the currency of the operation
- **Settlement currency** — the currency of actual settlement
- **Reporting currency** — the currency of consolidated reporting
- **Functional currency** — the operating currency of the legal entity

All monetary amounts are stored as integers in minor units. No floating point. No rounding errors.

### Multi-Accounting Framework

Accounting rules are not hardcoded. The financial engine is decoupled from the chart of accounts, tax rules, and compliance requirements.

An Accounting Framework in Axions360 is a complete, swappable rule set:

```
Accounting Framework
├── Chart of Accounts
├── Journal Types
├── Fiscal Rules
├── Tax Rules
├── Reporting Structure
├── Closing Rules
└── Regulatory Constraints
```

Designed to support: **SYSCOHADA · SYSCOA · IFRS · PCG · CPA Canada · Custom Frameworks**

The same financial engine, the same transaction table, the same tenant — different accounting frameworks, without touching the core.

### Multi-Country by Design

A single tenant can operate across multiple countries simultaneously. Country-specific rules — tax, compliance, currency, accounting framework — are resolved dynamically per store and per organization, not per deployment.

### Offline-First

Internet is optional. Business continuity is mandatory.

Synchro runs entirely on a local network. POS terminals, kitchen displays, inventory — everything works without internet. A Sync Engine handles conflict resolution when connectivity is available, with business-logic-aware merge rules.

### Payment Provider Agnostic

Axions360 does not depend on a specific payment provider. The platform integrates with any payment infrastructure through a unified abstraction layer:

- Mobile Money providers
- Banks and card processors
- Third-party payment gateways

including ATANAX financial infrastructure services.

### Activation, Not Migration

```
2026 — Install Synchro Restaurant
2027 — Activate Axions Finance      → transactions already there
2028 — Activate Axions HR           → employees already there
2029 — Activate Multi-site Reporting → all history available
```

No export. No import. No data loss. No downtime.

---

## The ATANAX Ecosystem

Axions360 is part of the ATANAX ecosystem.

```
ATANAX
│
├── Synchro                    Operations on the ground
│   ├── Synchro Restaurant     POS · KDS · Menus · Tables · Kitchen
│   ├── Synchro Hotel          Reservations · Rooms · Housekeeping
│   └── Synchro Commerce       POS · Stock · Inventory · Barcodes
│
├── Axions360                  Back-office and group management
│   ├── Axions Finance         Multi-framework accounting · Ledger · Consolidation
│   ├── Axions HR              Employees · Payroll · Leave · Contracts
│   ├── Axions CRM             Clients · Suppliers · Interactions
│   ├── Axions Projects        Tasks · Milestones · Teams
│   └── Axions Immobilier      Properties · Leases · Tenants
│
└── Dymmo                      Financial infrastructure platform
    └── atanax.com/dymmo
```

> Axions360 is designed to work independently of Dymmo. Any compliant payment provider can be integrated through the payment abstraction layer.

---

## How Axions360 Compares

| Capability                                 | Axions360            |
| ------------------------------------------ | -------------------- | 
| Offline POS                                | ✅ Native            | 
| Multi-Accounting Framework                 | ✅ Designed for it   |
| Mobile Money ready                         | ✅ Abstraction layer |
| OHADA / SYSCOHADA                          | ✅                   | 
| Database-native i18n architecture          | ✅                   | 
| Multi-currency (4 roles)                   | ✅                   | 
| Activation instead of migration            | ✅                   | 
| Open core                                  | ✅                   | 
| Built for Africa                           | ✅                   | 
| Framework-agnostic architectural contracts | ✅                   | 

---

## Open Core Model

The software is free. Revenue comes from what surrounds it.

| Free forever                  | Paid (cloud & services)                      |
| ----------------------------- | -------------------------------------------- |
| All Synchro modules           | Cloud synchronization                        |
| Multi-user, multi-store local | Multi-site dashboards                        |
| Full offline operation        | Automated backups                            |
| Standard reports              | Advanced analytics                           |
| Community support             | Consolidated group reporting                 |
|                               | Notification channels (SMS, WhatsApp, Email) |
|                               | API access                                   |
|                               | Priority support                             |

---

## Architecture

### Philosophy

> Start as a modular monolith. Code as if every module could become an independent service tomorrow.

Transactional consistency for the ERP core. Simple local deployment for Synchro offline. Clear extraction paths for high-concurrency services.

### Stack

| Layer               | Technology                   |
| ------------------- | ---------------------------- |
| Core API            | PHP 8.3+ (Laravel-based)     |
| Web App             | React 19 · TypeScript · Vite |
| Desktop (POS / KDS) | C++ · Tauri                  |
| Mobile              | React Native · Capacitor     |
| Database            | PostgreSQL 16                |
| Cache / Queues      | Redis                        |
| Analytics           | ClickHouse                   |
| Cryptography        | Rust                         |
| AI / Scoring        | Python · FastAPI             |
| Sites               | Astro                        |

### Architectural Contracts

| Contract                 | Purpose                                                                                |
| ------------------------ | -------------------------------------------------------------------------------------- |
| **Entity Contract V1**   | Versioned structural standard for every entity table                                   |
| **Entity Registry**      | Architectural rules declared in the database — not in framework code                   |
| **Opscodes**             | Universal operational language linking events, audit, notifications, and translations  |
| **Operation Codes**      | Financial impact catalogue (DEBIT / CREDIT / NEUTRAL)                                  |
| **Accounting Framework** | Complete accounting rule-set abstraction — chart of accounts, fiscal rules, compliance |
| **Lookup Tables**        | Zero hardcoded ENUMs — all statuses and types are extensible database tables with i18n |

---

## Roadmap

### Phase 1 — Kernel _(in progress)_

- [x] Architecture contracts
- [x] Entity Contract V1
- [ ] Axions Kernel — core schema
- [ ] Entity Registry
- [ ] Opscodes catalogue
- [ ] Authentication
- [ ] RBAC — roles and permissions
- [ ] Multi-language foundation

### Phase 2 — Synchro Restaurant

- [ ] POS · KDS · Menus · Tables · Inventory
- [ ] Local network sync
- [ ] Offline conflict resolution

### Phase 3 — Axions ERP Core

- [ ] Axions Finance — SYSCOHADA first
- [ ] Axions HR
- [ ] Axions CRM
- [ ] Multi-site reporting

### Phase 4 — Scale

- [ ] Synchro Hotel · Synchro Commerce
- [ ] Additional accounting frameworks
- [ ] Multi-country deployment
- [ ] Payment provider integrations

---

## For Developers

Axions360 is built on contracts, not conventions. Before contributing, read the three architecture documents:

| Document                 | Purpose                                                |
| ------------------------ | ------------------------------------------------------ |
| `ATANAX_ARCHITECTURE.md` | System design, stack decisions, evaluated technologies |
| `AXIONS_KERNEL.md`       | Core schema, entity contracts, module rules            |
| `AXIONS_SCHEMA_RULES.md` | Data conventions, i18n, opscodes, migrations           |

**Rules every contributor must follow:**

- Architectural rules live in the database — not in framework code
- No hardcoded ENUMs — Lookup Tables with i18n only
- Monetary amounts are always `BIGINT` minor units — never `DECIMAL` or `FLOAT`
- Public identifiers are always ULIDs — never integer IDs in URLs or API responses
- Modules communicate via Events (opscodes) — never by direct cross-module table access
- Every financial operation creates an entry in the core `transactions` table
- `audit_logs` and `domain_events` are append-only — no UPDATE, no DELETE

**We will not merge code that:**

- Introduces a hardcoded ENUM in a schema
- Bypasses tenant isolation
- Uses `DECIMAL` or `FLOAT` for monetary values
- Accesses another module's tables directly

---

## Join the Journey

Axions360 is being built openly for Africa.

- **General:** [hello@atanax.com](mailto:hello@atanax.com)
- **Developers:** [dev@atanax.com](mailto:dev@atanax.com)
- **Partnerships:** [partners@atanax.com](mailto:partners@atanax.com)
- **Community:** [GitHub Discussions](https://github.com/atanaxinc/axions360/discussions)

---

## License

```
Code                              Services
────────────────────────────────  ────────────────────────────────
Axions Kernel                     Managed Hosting
Synchro (Restaurant, Hotel,       Cloud Synchronization
  Commerce)                       Premium Support
Axions ERP (Finance, HR, CRM,     Training & Certification
  Projects, Immobilier)           Marketplace Services
Public Documentation              Future Premium Integrations

AGPL-3.0                          Commercial Agreements
```

Axions360 Community Edition is released under the **GNU Affero General Public License v3.0 (AGPL-3.0)**.

Commercial services such as managed hosting, cloud synchronization, training, certification, premium support, and certain integrations may be offered under separate commercial agreements.

See the [LICENSE](./LICENSE) file for details.

---

## About ATANAX

ATANAX Inc. is a Montreal-based technology company building sovereign digital infrastructure for Africa.

**Built in Africa. Ready for the World.**

[atanax.com](https://atanax.com) · [LinkedIn](https://linkedin.com/company/atanax) · [GitHub](https://github.com/atanaxinc)

---

<div align="center">

**Built in Africa. Ready for the World.**

</div>

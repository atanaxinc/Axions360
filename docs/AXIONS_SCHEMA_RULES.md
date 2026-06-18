# AXIONS SCHEMA RULES — Conventions de Conception des Données

> **Ce document est le contrat de conception des schémas de données pour tout l'écosystème Axions360.**
> À lire conjointement avec `AXIONS_KERNEL.md` (quoi créer) et `ATANAX_ARCHITECTURE.md` (comment déployer).
> Ces règles s'appliquent à **tous les modules et tous les services** : Synchro, Axions ERP, Dymmo, Integration Layer.
> Elles sont **agnostiques au framework** — elles s'appliquent que le service soit Laravel, Spring Boot, Go, Rust ou C++.

---

## Table des matières

1. [Principe fondamental — Agnosticisme Technologique](#1-principe-fondamental--agnosticisme-technologique)
2. [Entity Contract V1 — Structure d'une table entité](#2-entity-contract-v1--structure-dune-table-entité)
3. [Entity Registry — Le tier de criticité](#3-entity-registry--le-tier-de-criticité)
4. [Lookup Tables — Zéro ENUM hardcodé](#4-lookup-tables--zéro-enum-hardcodé)
5. [Internationalisation (i18n) — Pattern Sidecar](#5-internationalisation-i18n--pattern-sidecar)
6. [Système de langues — Activation par Tenant/Store](#6-système-de-langues--activation-par-tenantstore)
7. [Opscodes — Le langage universel du système](#7-opscodes--le-langage-universel-du-système)
8. [Operation Codes — L'impact financier (Dymmo/Finance)](#8-operation-codes--limpact-financier-dymmofinance)
9. [Règles de protection des données (PII)](#9-règles-de-protection-des-données-pii)
10. [Verrouillage optimiste — row_version](#10-verrouillage-optimiste--row_version)
11. [Règles de migrations](#11-règles-de-migrations)
12. [Idempotence](#12-idempotence)
13. [Ce qui est interdit](#13-ce-qui-est-interdit)
14. [Checklist avant chaque PR](#14-checklist-avant-chaque-pr)

---

## 1. Principe fondamental — Agnosticisme Technologique

### 1.1 La règle

> **Toute règle architecturale qui doit survivre à un changement de framework ou de langage doit vivre dans la base de données, pas dans le code applicatif.**

Le code applicatif **lit** les règles. Il ne les **définit** pas.

### 1.2 Pourquoi cette règle est critique pour ATANAX

L'écosystème ATANAX est intentionnellement multi-technologique :

```
Laravel      → Axions Core, Synchro Cloud
Go           → Dymmo, API Gateway, Sync Engine
Spring Boot  → Reporting bancaire, ERP Enterprise
Rust         → Sécurité, cryptographie, STC
C++          → KDS, POS Desktop offline
Python       → Quotient Engine, IA
```

Si une règle d'architecture est codée dans un `BaseModel` Laravel, elle disparaît dès qu'un service Go ou Spring Boot accède à la même table. C'est une fausse sécurité.

### 1.3 Application concrète

| ❌ Règle couplée au framework | ✅ Règle agnostique |
|-------------------------------|---------------------|
| `BaseModel` Laravel impose `row_version` | `entity_registry` déclare `has_row_version = true` — tout service lit ce registre |
| Observer Laravel pour l'audit | `entity_registry.has_audit = true` — tout service sait qu'il doit auditer |
| Soft delete via `SoftDeletes` trait | `entity_registry.has_soft_delete = true` — tout service respecte `deleted_at` |
| ENUM dans le schéma | Lookup Table en base — tout service lit la même table |
| Texte traduit dans les fichiers `.po` du framework | Traductions en base via Pattern Sidecar — tout service accède aux mêmes traductions |
| Constante de statut dans le code | Code dans `{entity}_statuses` — tout service lit la même table |

### 1.4 Comment un service lit les règles

Au démarrage, tout service charge le registre des entités qu'il manipule :

```go
// Go — Dymmo lit entity_registry au démarrage
config := registry.Load("transactions")
// config.HasRowVersion = true
// config.HasAudit = true
// config.Tier = 1
```

```java
// Spring Boot — même source
EntityConfig config = entityRegistry.find("employees");
// config.isHasRowVersion() == true
```

```php
// Laravel — même source
$config = EntityRegistry::find('orders');
// $config->has_row_version == true
```

**La source est unique. Le comportement est identique sur tous les services.**

---

## 2. Entity Contract V1 — Structure d'une table entité

### 2.1 Définition

L'**Entity Contract V1** définit les colonnes contractuellement obligatoires sur toute table entité du système Axions360.

Une table entité est une table qui représente un objet métier avec un cycle de vie propre (création, modification, archivage).

### 2.2 Colonnes obligatoires

```sql
-- Identifiant interne — usage SQL uniquement, jamais exposé
id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY

-- Identifiant public — usage API, URLs, références externes
ulid        CHAR(26) UNIQUE NOT NULL

-- Isolation souveraine — toute donnée appartient à exactement un tenant
tenant_id   BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)

-- Statut externalisé — jamais ENUM hardcodé (voir section 4)
status_code VARCHAR(30) NOT NULL REFERENCES {entity}_statuses(code)

-- Compteur de génération de ligne — verrouillage optimiste (voir section 10)
-- Présence conditionnelle selon entity_registry.has_row_version (voir section 3)
row_version INTEGER NOT NULL DEFAULT 0

-- Timestamps avec timezone explicite — toujours UTC
created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
```

### 2.3 Rôle de chaque colonne

| Colonne | Rôle | Pourquoi |
|---------|------|----------|
| `id` | Clé primaire interne | Performance des JOINs SQL. Jamais exposé à l'extérieur. |
| `ulid` | Identifiant public ordonné | Ordonné dans le temps, URL-safe, opaque. Protège contre l'énumération. |
| `tenant_id` | Isolation multi-tenant | Toute donnée appartient à exactement un tenant. Scope automatique. |
| `status_code` | Cycle de vie externalisé | Extensible sans migration, traduisible, agnostique au framework. |
| `row_version` | Verrouillage optimiste | Protège contre les écritures concurrentes silencieuses. Conditionnel selon le tier. |
| timestamps UTC | Traçabilité temporelle | `WITH TIME ZONE` — jamais de timezone implicite sur un système multi-pays. |

### 2.4 Exceptions documentées

| Table | Exception | Raison |
|-------|-----------|--------|
| `tenants` | Pas de `tenant_id` | Elle EST la racine — exception G-ROOT-01 |
| Lookup tables globales | Pas de `tenant_id`, pas de `row_version` | Données système partagées, pas de cycle de vie client |
| `audit_logs`, `domain_events` | Pas de `row_version`, pas de `updated_at` | Append-only — la modification est interdite par définition |
| Tables pivot simples | Pas de `ulid` | Non exposées en API — `id` interne suffit |

### 2.5 Versioning du contrat

L'Entity Contract est versionné. Toute table indique sa conformité dans `entity_registry.contract_version`.

```
Entity Contract V1  → colonnes définies en section 2.2 (Juin 2026)
Entity Contract V2  → évolution future documentée avec plan de migration
```

Quand le contrat évolue, les tables existantes sont migrées progressivement avec un plan documenté et approuvé. Les nouvelles tables doivent respecter la version courante du contrat.

---

## 3. Entity Registry — Le tier de criticité

### 3.1 Définition

L'`entity_registry` est une table Core du Kernel. Elle déclare chaque entité du système, son tier de criticité, et les protections qui s'y appliquent.

C'est la **source unique de vérité** pour les règles d'architecture des entités. Tout service — Laravel, Go, Spring Boot, Rust — consulte cette table pour savoir comment traiter une entité donnée.

### 3.2 Schéma

```sql
CREATE TABLE entity_registry (
    entity_name       VARCHAR(100) PRIMARY KEY,  -- 'employees', 'transactions', 'orders'
    module_slug       VARCHAR(50)  NOT NULL,      -- 'axions-kernel', 'synchro-restaurant'
    contract_version  SMALLINT     NOT NULL DEFAULT 1, -- version de l'Entity Contract
    tier              SMALLINT     NOT NULL,      -- 1, 2, 3, 4 (voir section 3.3)
    has_row_version   BOOLEAN      NOT NULL DEFAULT false,
    has_audit         BOOLEAN      NOT NULL DEFAULT false,
    has_soft_delete   BOOLEAN      NOT NULL DEFAULT false,
    has_idempotency   BOOLEAN      NOT NULL DEFAULT false,
    has_pii           BOOLEAN      NOT NULL DEFAULT false,
    description       TEXT         NULL,
    created_at        TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 3.3 Les quatre tiers

```
TIER 1 — CRITICAL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Protections actives :
  ✓ row_version         verrouillage optimiste obligatoire
  ✓ audit               toute modification tracée dans audit_logs
  ✓ soft_delete         jamais de DELETE physique — deleted_at uniquement
  ✓ idempotency         sur les opérations sensibles
  ✓ pii                 chiffrement si données personnelles présentes

Exemples :
  employees, transactions, documents, contracts, wallets,
  kyc_profiles, compliance_records

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TIER 2 — STANDARD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Protections actives :
  ✓ row_version         verrouillage optimiste
  ✓ audit               audit recommandé sur les actions sensibles
  ✓ soft_delete         recommandé

Exemples :
  orders, invoices, contacts, store_settings, reservations,
  menu_items, products

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TIER 3 — OPERATIONAL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Protections actives :
  ✗ row_version         pas nécessaire
  ✗ audit               pas d'audit systématique
  ✗ soft_delete         DELETE physique autorisé

Exemples :
  sessions, push_tokens, temporary_sync_queue,
  notification_logs, cache_entries, webhooks_logs

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TIER 4 — REFERENCE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Protections actives :
  ✗ row_version         pas de concurrence sur des données système
  ✗ audit               données immuables par nature
  ✗ soft_delete         remplacement par nouvelle version si nécessaire

Exemples :
  lookup tables, languages, opscodes, operation_codes,
  countries, currencies, timezones
```

### 3.4 Données de référence (Seeder)

```sql
INSERT INTO entity_registry
    (entity_name, module_slug, tier, has_row_version, has_audit, has_soft_delete, has_idempotency, has_pii)
VALUES
-- Tier 1 — Critical
('employees',         'axions-kernel',       1, true,  true,  true,  false, true),
('transactions',      'axions-kernel',       1, true,  true,  false, true,  false),
('documents',         'axions-kernel',       1, true,  true,  true,  false, false),
('people',            'axions-kernel',       1, true,  true,  true,  false, true),
('users',             'axions-kernel',       1, true,  true,  true,  false, false),

-- Tier 2 — Standard
('contacts',          'axions-kernel',       2, true,  true,  true,  false, false),
('stores',            'axions-kernel',       2, true,  true,  true,  false, false),
('organizations',     'axions-kernel',       2, true,  true,  true,  false, false),
('restaurant_orders', 'synchro-restaurant',  2, true,  true,  true,  false, false),
('hotel_reservations','synchro-hotel',       2, true,  true,  true,  false, false),

-- Tier 3 — Operational
('notification_logs', 'axions-kernel',       3, false, false, false, false, false),
('sync_queue',        'axions-kernel',       3, false, false, false, false, false),

-- Tier 4 — Reference
('languages',         'axions-kernel',       4, false, false, false, false, false),
('opscodes',          'axions-kernel',       4, false, false, false, false, false),
('operation_codes',   'axions-kernel',       4, false, false, false, false, false);
```

### 3.5 Comment un service utilise entity_registry

```go
// Go (Dymmo) — lecture au démarrage
func (s *TransactionService) Save(tx Transaction) error {
    cfg := s.registry.Get("transactions")

    if cfg.HasRowVersion {
        // appliquer le verrouillage optimiste
    }
    if cfg.HasAudit {
        // enregistrer dans audit_logs
    }
    if cfg.HasIdempotency {
        // vérifier l'idempotency_key avant insertion
    }
}
```

```java
// Spring Boot — même logique
@Service
public class EmployeeService {
    private final EntityRegistry registry;

    public Employee update(UpdateEmployeeDTO dto) {
        EntityConfig cfg = registry.get("employees");

        if (cfg.isHasRowVersion()) {
            // verrouillage optimiste
        }
        if (cfg.isHasSoftDelete()) {
            // ne jamais appeler repository.delete() — utiliser archived_at
        }
    }
}
```

```php
// Laravel — même logique, même source
$cfg = EntityRegistry::find('orders');
// $cfg->tier == 2
// $cfg->has_row_version == true
```

---

## 4. Lookup Tables — Zéro ENUM hardcodé

### 4.1 Règle d'or — Invariant G1 (No-Hardcode)

**Aucun statut, type ou catégorie ne doit être codé en dur dans le schéma (ENUM, CHECK constraint) ou dans le code applicatif.**

Raison : un ENUM hardcodé est couplé au schéma. Ajouter une valeur requiert une migration ALTER TABLE en production. Une Lookup Table permet d'ajouter une valeur via un simple INSERT, sans toucher au schéma, sans downtime, et avec traduction automatique.

```sql
-- ❌ INTERDIT — ENUM hardcodé dans le schéma
status ENUM('active','suspended','deleted') NOT NULL

-- ✅ CORRECT — référence vers une Lookup Table
status_code VARCHAR(30) NOT NULL REFERENCES order_statuses(code)
```

### 4.2 Structure d'une Lookup Table

```sql
CREATE TABLE order_statuses (
    code        VARCHAR(30) PRIMARY KEY,    -- 'PENDING', 'COMPLETED', 'CANCELLED'
    is_active   BOOLEAN  NOT NULL DEFAULT true,
    sort_order  SMALLINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Table i18n sidecar obligatoire (voir section 5)
CREATE TABLE order_status_i18n (
    parent_code   VARCHAR(30) NOT NULL REFERENCES order_statuses(code) ON DELETE CASCADE,
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    label         TEXT NOT NULL,
    description   TEXT NULL,
    PRIMARY KEY (parent_code, language_code)
);
```

### 4.3 Convention de nommage

Les codes de lookup sont en `SCREAMING_SNAKE_CASE` — stables, lisibles, agnostiques à toute langue :

```
order_statuses          → PENDING, PROCESSING, COMPLETED, CANCELLED, REFUNDED
employee_statuses       → ACTIVE, ON_LEAVE, SUSPENDED, TERMINATED
document_types          → INVOICE, RECEIPT, CONTRACT, QUOTE, REPORT
store_types             → RESTAURANT, HOTEL, COMMERCE, REAL_ESTATE, AGENCY
employment_types        → FULL_TIME, PART_TIME, CONTRACTOR, INTERN, DAILY_WORKER
transaction_statuses    → PENDING, COMPLETED, FAILED, CANCELLED, REVERSED
```

### 4.4 La colonne dans la table principale porte le suffixe `_code`

```sql
-- Dans restaurant_orders
status_code     VARCHAR(30) NOT NULL REFERENCES order_statuses(code)
order_type_code VARCHAR(30) NOT NULL REFERENCES order_types(code)

-- Dans employees
status_code          VARCHAR(30) NOT NULL REFERENCES employee_statuses(code)
employment_type_code VARCHAR(30) NOT NULL REFERENCES employment_types(code)
```

---

## 5. Internationalisation (i18n) — Pattern Sidecar

### 5.1 Principe

Tout texte affiché à l'utilisateur est traduit en base de données — pas dans des fichiers de traduction couplés à un framework.

**Raison** : si les traductions vivent dans `resources/lang/fr.json` (Laravel) ou `messages_fr.properties` (Spring Boot), elles sont inaccessibles aux autres services. Un service Go qui génère une notification ne peut pas lire les fichiers Laravel.

```
Traduit (en base)                   Jamais traduit (stable en anglais)
─────────────────────────────────   ─────────────────────────────────
Labels de statuts                   tenant_id, store_id
Noms de modules (affichage)         module_slug, permission_slug
Labels de rôles et permissions      event_type, opscode code
Noms de produits et menus           API routes
Templates de notifications          Noms de tables et colonnes
Messages d'erreur (UI)              Codes techniques (PENDING, etc.)
Contenu de documents clients
```

### 5.2 Pattern Table Sidecar

```sql
CREATE TABLE {table_name}_i18n (
    parent_code   VARCHAR(30) NOT NULL,
    language_code VARCHAR(10) NOT NULL,
    label         TEXT NOT NULL,
    description   TEXT NULL,
    PRIMARY KEY (parent_code, language_code),

    -- CASCADE sur parent — supprimer un code supprime ses traductions
    CONSTRAINT fk_{table}_i18n_parent
        FOREIGN KEY (parent_code) REFERENCES {table_name}(code) ON DELETE CASCADE,

    -- Pas de CASCADE sur language — les langues ne se suppriment pas
    CONSTRAINT fk_{table}_i18n_lang
        FOREIGN KEY (language_code) REFERENCES languages(code)
);
```

### 5.3 Entités métier multilingues

Pour les entités dont le contenu lui-même est multilingue (produits, menus, catégories) :

```sql
-- Table principale — données techniques stables
CREATE TABLE restaurant_menu_items (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    ulid        CHAR(26) UNIQUE NOT NULL,
    tenant_id   BIGINT UNSIGNED NOT NULL REFERENCES tenants(id),
    store_id    BIGINT UNSIGNED NOT NULL REFERENCES stores(id),
    price       BIGINT NOT NULL,            -- sous-unités
    status_code VARCHAR(30) NOT NULL REFERENCES menu_item_statuses(code),
    row_version INTEGER NOT NULL DEFAULT 0,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Table sidecar pour le contenu traduit
CREATE TABLE restaurant_menu_item_translations (
    menu_item_id  BIGINT UNSIGNED NOT NULL REFERENCES restaurant_menu_items(id) ON DELETE CASCADE,
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    name          VARCHAR(255) NOT NULL,
    description   TEXT NULL,
    PRIMARY KEY (menu_item_id, language_code)
);
```

### 5.4 Nommage

| Type | Convention | Exemple |
|------|-----------|---------|
| Lookup statique | `{lookup_table}_i18n` | `order_status_i18n` |
| Entité métier | `{entity_table}_translations` | `menu_item_translations` |
| Opscodes | `opscode_translations` | — |
| Templates | `notification_template_translations` | — |

---

## 6. Système de langues — Activation par Tenant/Store

### 6.1 Table `languages` — globale

```sql
CREATE TABLE languages (
    code        VARCHAR(10) PRIMARY KEY,    -- BCP 47 : 'fr', 'en', 'ar', 'ff'
    name        VARCHAR(100) NOT NULL,      -- "Français"
    native_name VARCHAR(100) NOT NULL,      -- nom dans la langue elle-même
    direction   CHAR(3) NOT NULL DEFAULT 'ltr',  -- 'ltr' ou 'rtl'
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO languages VALUES
    ('fr',  'French',   'Français',  'ltr', true),
    ('en',  'English',  'English',   'ltr', true),
    ('ar',  'Arabic',   'العربية',   'rtl', true),
    ('ff',  'Fula',     'Pulaar',    'ltr', true),
    ('man', 'Mandinka', 'Mandingo',  'ltr', true),
    ('sus', 'Susu',     'Soussou',   'ltr', true),
    ('wo',  'Wolof',    'Wolof',     'ltr', true),
    ('dyu', 'Dyula',    'Dioula',    'ltr', true);
```

### 6.2 Activation par tenant et par store

```sql
CREATE TABLE tenant_languages (
    tenant_id     BIGINT UNSIGNED NOT NULL REFERENCES tenants(id),
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    is_default    BOOLEAN NOT NULL DEFAULT false,
    is_active     BOOLEAN NOT NULL DEFAULT true,
    activated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (tenant_id, language_code)
);

CREATE TABLE store_languages (
    store_id      BIGINT UNSIGNED NOT NULL REFERENCES stores(id),
    tenant_id     BIGINT UNSIGNED NOT NULL REFERENCES tenants(id),
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    is_default    BOOLEAN NOT NULL DEFAULT false,
    is_active     BOOLEAN NOT NULL DEFAULT true,
    PRIMARY KEY (store_id, language_code)
);
```

### 6.3 Règle de résolution

```
1. Langue préférée de l'utilisateur  (people.preferred_locale)
2. Langue par défaut du store         (store_languages.is_default)
3. Langue par défaut du tenant        (tenant_languages.is_default)
4. Français ('fr')                    ← fallback ultime du système
```

---

## 7. Opscodes — Le langage universel du système

### 7.1 Définition

Un opscode est une **clé opérationnelle stable** qui identifie une action métier dans le système. C'est le vocabulaire interne universel d'Axions360, indépendant de tout framework ou service.

```
opscode          → ce qui s'est passé (événement métier)
operation_code   → quel impact financier cela produit (section 8)
```

Les opscodes relient proprement : Events · Audit logs · Notifications · Traductions · Reporting · Automations · Intégrations externes.

### 7.2 Schéma

```sql
CREATE TABLE opscode_domains (
    code       VARCHAR(30) PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE opscode_domain_i18n (
    parent_code   VARCHAR(30) NOT NULL REFERENCES opscode_domains(code) ON DELETE CASCADE,
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    label         TEXT NOT NULL,
    PRIMARY KEY (parent_code, language_code)
);

CREATE TABLE opscode_severities (
    code       VARCHAR(20) PRIMARY KEY,   -- INFO, NOTICE, WARNING, ERROR, CRITICAL, FATAL
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE opscode_severity_i18n (
    parent_code   VARCHAR(20) NOT NULL REFERENCES opscode_severities(code) ON DELETE CASCADE,
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    label         TEXT NOT NULL,
    PRIMARY KEY (parent_code, language_code)
);

CREATE TABLE opscodes (
    id                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    code              VARCHAR(128) UNIQUE NOT NULL,  -- 'synchro.restaurant.order.cancelled'
    module_slug       VARCHAR(50)  NOT NULL,
    domain_code       VARCHAR(30)  NOT NULL REFERENCES opscode_domains(code),
    category          VARCHAR(30)  NOT NULL,
    action            VARCHAR(30)  NOT NULL,
    severity_code     VARCHAR(20)  NOT NULL REFERENCES opscode_severities(code),
    is_auditable      BOOLEAN NOT NULL DEFAULT true,
    is_notifiable     BOOLEAN NOT NULL DEFAULT false,
    is_permissionable BOOLEAN NOT NULL DEFAULT false,
    created_at        TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE opscode_translations (
    opscode_id       BIGINT UNSIGNED NOT NULL REFERENCES opscodes(id) ON DELETE CASCADE,
    language_code    VARCHAR(10) NOT NULL REFERENCES languages(code),
    label            VARCHAR(255) NOT NULL,
    description      TEXT NULL,
    message_template TEXT NULL,   -- "La commande #{order_number} a été annulée par {user_name}."
    PRIMARY KEY (opscode_id, language_code)
);
```

### 7.3 Convention de nommage des codes

```
Format : {module}.{domaine}.{action}

synchro.restaurant.order.created
synchro.restaurant.order.cancelled
synchro.restaurant.order.paid
synchro.hotel.reservation.confirmed
synchro.commerce.stock.adjusted
axions.hr.employee.hired
axions.hr.employee.terminated
axions.finance.transaction.recorded
axions.auth.user.login_failed
axions.system.module.activated
axions.system.sync.conflict_detected
dymmo.payment.transfer.confirmed
dymmo.payment.transfer.failed
dymmo.compliance.account.frozen
```

### 7.4 Domaines standards

```
AUTH        → Authentification, sessions, mots de passe
ORDER       → Commandes, transactions POS
INVENTORY   → Stock, ajustements, transferts
HR          → Employés, contrats, congés
PAYMENT     → Paiements, remboursements
DOCUMENT    → Factures, contrats, signatures
DEVICE      → KDS, POS hardware, enregistrement d'appareils
SYNC        → Synchronisation offline/online, conflits
COMPLIANCE  → Gel de compte, saisie, conformité réglementaire
SYSTEM      → Modules, tenants, mises à jour système
CRM         → Clients, contacts, interactions
```

---

## 8. Operation Codes — L'impact financier (Dymmo/Finance)

### 8.1 Distinction

```
opscode          = raconte l'événement métier
operation_code   = décrit l'impact sur le grand livre financier

synchro.restaurant.order.paid  →  MERCHANT_PAYMENT_CREDIT  (wallet marchand crédité)
                               →  MERCHANT_FEE              (commission Dymmo débitée)
```

### 8.2 Schéma

```sql
CREATE TABLE operation_domains (
    code       VARCHAR(30) PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE operation_domain_i18n (
    parent_code   VARCHAR(30) NOT NULL REFERENCES operation_domains(code) ON DELETE CASCADE,
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    label         TEXT NOT NULL,
    PRIMARY KEY (parent_code, language_code)
);

CREATE TABLE operation_impacts (
    code       VARCHAR(20) PRIMARY KEY,    -- DEBIT, CREDIT, NEUTRAL
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE operation_impact_i18n (
    parent_code   VARCHAR(20) NOT NULL REFERENCES operation_impacts(code) ON DELETE CASCADE,
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    label         TEXT NOT NULL,
    PRIMARY KEY (parent_code, language_code)
);

CREATE TABLE operation_codes (
    code          VARCHAR(50) PRIMARY KEY,
    domain_code   VARCHAR(30) NOT NULL REFERENCES operation_domains(code),
    impact_code   VARCHAR(20) NOT NULL REFERENCES operation_impacts(code),
    is_reversible BOOLEAN NOT NULL DEFAULT true,
    created_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE operation_code_i18n (
    parent_code   VARCHAR(50) NOT NULL REFERENCES operation_codes(code) ON DELETE CASCADE,
    language_code VARCHAR(10) NOT NULL REFERENCES languages(code),
    label         TEXT NOT NULL,
    description   TEXT NULL,
    PRIMARY KEY (parent_code, language_code)
);
```

### 8.3 Catalogue de référence

```
TRANSFER :
  P2P_DEBIT              DEBIT    reversible
  P2P_CREDIT             CREDIT   reversible
  REMITTANCE_DEBIT       DEBIT    reversible
  REMITTANCE_CREDIT      CREDIT   reversible

PAYMENT :
  MERCHANT_PAYMENT_DEBIT   DEBIT    reversible
  MERCHANT_PAYMENT_CREDIT  CREDIT   reversible
  CARD_PAYMENT_DEBIT       DEBIT    reversible
  CARD_REFUND_CREDIT       CREDIT   non-reversible

FEE :
  P2P_FEE                DEBIT    non-reversible
  REMITTANCE_FEE         DEBIT    non-reversible
  MERCHANT_FEE           DEBIT    non-reversible
  SERVICE_FEE_DEBIT      DEBIT    non-reversible
  SERVICE_FEE_REVENUE    CREDIT   non-reversible

FX :
  FX_CONVERSION_DEBIT    DEBIT    non-reversible
  FX_CONVERSION_CREDIT   CREDIT   non-reversible
  FX_SPREAD_FEE          DEBIT    non-reversible

REWARD :
  REWARD_CREDIT          CREDIT   non-reversible
  REWARD_REVERSAL        DEBIT    non-reversible

COMPLIANCE :
  COMPLIANCE_FREEZE      NEUTRAL  reversible
  COMPLIANCE_SEIZURE     DEBIT    non-reversible
  COMPLIANCE_RESTITUTION CREDIT   non-reversible

OFFLINE :
  OFFLINE_PAYMENT_DEBIT  DEBIT    reversible
  OFFLINE_PAYMENT_CREDIT CREDIT   reversible
  OFFLINE_SYNC_SETTLE    NEUTRAL  non-reversible
```

---

## 9. Règles de protection des données (PII)

### 9.1 Chiffrement AES-256-GCM

Toute donnée personnelle sensible est chiffrée **au niveau applicatif** avant insertion. Le chiffrement se fait dans le service, pas dans la base — pour que tout service puisse déchiffrer avec la bonne clé.

```sql
id_number_encrypted    TEXT NOT NULL        -- AES-256-GCM, jamais en clair
id_number_blind_index  VARCHAR(64) NOT NULL -- HMAC-SHA256 pour la recherche
key_version_id         BIGINT UNSIGNED NULL REFERENCES kyc_key_versions(id)
```

### 9.2 Blind Index — recherche sans déchiffrement

```sql
-- ✅ Recherche sur données chiffrées
SELECT * FROM people
WHERE id_number_blind_index = HMAC(?, 'sha256', ?)
```

### 9.3 Clé KMS — règle absolue

```sql
-- ✅ Seule la référence KMS est stockée
key_ref  VARCHAR(500) NOT NULL  -- 'arn:aws:kms:eu-west-1:123:key/abc-123'

-- ❌ Jamais
encryption_key  VARCHAR(255)
```

### 9.4 Masquage dans les logs

Toute implémentation dans n'importe quel service doit masquer les PII dans les logs. La convention est indépendante du langage :

```
RÈGLE : Ne jamais logger les champs marqués has_pii = true dans entity_registry.
CONVENTION : Remplacer par "***REDACTED***" dans tous les logs structurés.
```

---

## 10. Verrouillage optimiste — row_version

### 10.1 Pourquoi `row_version` et pas `version`

`version` est ambigu dans un ERP — il peut désigner la version métier d'un document, la version d'un module, ou la version d'un contrat. `row_version` désigne sans ambiguïté le **compteur de génération de la ligne SQL**.

### 10.2 Pourquoi `INTEGER DEFAULT 0`

C'est un compteur d'incréments, pas un identifiant. Toute entité commence à `0` à sa création. La première modification la passe à `1`. Lisible, prévisible, sans ambiguïté.

### 10.3 Problème résolu

```
Utilisateur A lit la commande   → row_version = 3
Utilisateur B lit la commande   → row_version = 3

Utilisateur A sauvegarde        → row_version devient 4
Utilisateur B sauvegarde        → il vérifie row_version = 3, trouve 4 → ERREUR EXPLICITE
```

Sans `row_version`, la sauvegarde de B écrase silencieusement celle de A. Corruption silencieuse.

### 10.4 Application — agnostique au framework

La logique est identique dans tous les services :

```sql
-- Toujours vérifier et incrémenter atomiquement
UPDATE {table}
SET    {champs}, row_version = row_version + 1
WHERE  id = ? AND row_version = ?;

-- Si 0 lignes affectées → conflit détecté → retourner une erreur explicite
```

```go
// Go
result, _ := db.Exec(
    "UPDATE orders SET status_code=?, row_version=row_version+1 WHERE id=? AND row_version=?",
    newStatus, id, currentVersion,
)
if result.RowsAffected == 0 {
    return ErrOptimisticLock
}
```

```java
// Spring Boot — JPA gère nativement avec @Version
@Version
private Integer rowVersion;
```

```php
// Laravel — manuel mais identique
$affected = DB::table('orders')
    ->where('id', $id)
    ->where('row_version', $currentVersion)
    ->update(['status_code' => $status, 'row_version' => DB::raw('row_version + 1')]);

if ($affected === 0) throw new OptimisticLockException();
```

### 10.5 Présence conditionnelle

`row_version` n'est pas sur toutes les tables. Sa présence est déterminée par `entity_registry.has_row_version`. Les tables Tier 3 et Tier 4 n'en ont pas besoin.

---

## 11. Règles de migrations

### 11.1 Une migration = une table principale

Chaque fichier migration ne crée ou ne modifie qu'une seule table principale. La table i18n sidecar est la seule exception (elle n'a pas d'existence indépendante de sa table parent).

```
-- ✅ Correct
2026_06_15_000001_create_order_statuses_table    → order_statuses + order_status_i18n
2026_06_15_000002_create_order_types_table       → order_types + order_type_i18n

-- ❌ Interdit
Un fichier qui crée order_statuses ET order_types
```

### 11.2 Zéro ALTER TABLE pour colonnes avant production

Avant production, toute colonne manquante est ajoutée dans le fichier `create_TABLE` d'origine.

**Seule exception** : les migrations de FK déférées (contrainte uniquement, pas de colonne).

### 11.3 Pattern FK Déférée

Quand A (ancienne migration) doit référencer B (nouvelle migration) :

```
Étape 1 — migration de A : colonne déclarée sans FK (B n'existe pas encore)
Étape 2 — nouvelle migration séparée : FK posée après création de B
          down() supprime uniquement la FK — jamais la colonne
```

**Une migration FK déférée = une seule FK.**

### 11.4 Triptyque Modèle–Migration–Seeder

Toute Lookup Table suit ce triptyque :

```
{Entity}.php                          ← Modèle
YYYY_MM_DD_create_{entity}_table      ← Migration
{Entity}Seeder.php                    ← Seeder avec données de référence
```

Les tables entités (Entity Contract) n'ont généralement pas de Seeder.

### 11.5 Idempotence des migrations

```sql
-- Toujours utiliser les guards d'existence
CREATE TABLE IF NOT EXISTS order_statuses (...);

-- Vérifier avant d'ajouter une colonne
ALTER TABLE restaurant_orders ADD COLUMN IF NOT EXISTS notes TEXT;
```

---

## 12. Idempotence

Toute opération monétaire ou de sécurité doit inclure une `idempotency_key` avec contrainte `UNIQUE`.

```sql
idempotency_key  VARCHAR(128) NOT NULL,
CONSTRAINT uq_idempotency UNIQUE (tenant_id, idempotency_key)
```

La présence de cette contrainte est déclarée dans `entity_registry.has_idempotency`. Tout service qui écrit sur une entité avec `has_idempotency = true` doit vérifier l'idempotency avant insertion.

---

## 13. Ce qui est interdit

| ❌ Interdit | ✅ Correct | Raison |
|-------------|-----------|--------|
| Règle d'architecture dans le code applicatif uniquement | `entity_registry` en base | Survit au changement de framework |
| ENUM hardcodé dans le schéma | Lookup Table + `_code` + i18n | Extensible sans migration, traduisible |
| Traductions dans des fichiers framework | Pattern Sidecar en base | Accessible à tous les services |
| `version` comme nom de colonne de concurrence | `row_version` | Sans ambiguïté dans un ERP |
| `row_version` sur une table Tier 3 ou 4 | Selon `entity_registry.has_row_version` | Inutile sur les tables opérationnelles |
| DECIMAL ou FLOAT pour les montants | BIGINT sous-unités | Précision absolue, zéro arrondi flottant |
| Données PII en clair en base | AES-256-GCM + blind index | Conformité, sécurité |
| Clé cryptographique en base | `key_ref` KMS uniquement | La clé ne doit jamais persister |
| Données personnelles dans les logs | `***REDACTED***` sur champs `has_pii` | Vie privée, conformité |
| ALTER TABLE pour ajouter une colonne avant prod | Modifier le CREATE d'origine | Migrations propres |
| Deux tables dans une migration | Une migration = une table principale | Clarté, rollback propre |
| FK vers table inexistante au moment de la migration | Pattern FK Déférée | Ordre de migration garanti |
| UPDATE ou DELETE sur `audit_logs`, `domain_events` | Append-only absolu | Intégrité de l'historique |
| Opération financière sans `idempotency_key` | Contrainte UNIQUE déclarée dans entity_registry | Prévention des doublons |
| `id` entier exposé en API ou URL | `ulid` en externe | Sécurité, opacité |

---

## 14. Checklist avant chaque PR

**Schema**
- [ ] Entity Contract V1 respecté (id, ulid, tenant_id, status_code via Lookup, row_version si Tier 1/2, timestamps WITH TIME ZONE)
- [ ] Tier déclaré dans `entity_registry` avec les protections correctes
- [ ] Aucun ENUM hardcodé — Lookup Table avec i18n sidecar
- [ ] `row_version` présent uniquement sur les entités où `entity_registry.has_row_version = true`
- [ ] Montants en BIGINT sous-unités

**Internationalisation**
- [ ] Contenu visible utilisateur : table `_translations` ou `_i18n`
- [ ] Langues référencées via `languages(code)`, jamais des strings libres
- [ ] Traductions FR et EN au minimum sur toute nouvelle entrée i18n

**Opscodes & Operation Codes**
- [ ] Nouveaux événements : opscode déclaré dans le catalogue avec traductions FR/EN
- [ ] Nouvelles opérations financières : `operation_code` dans le catalogue

**Sécurité**
- [ ] PII chiffrées avec `blind_index` et `key_version_id`
- [ ] Aucune clé cryptographique en base — `key_ref` KMS uniquement
- [ ] Opérations monétaires : `idempotency_key` avec contrainte UNIQUE
- [ ] `id` entier non exposé en API — `ulid` en externe

**Migrations**
- [ ] Une migration = une table principale
- [ ] Zéro ALTER TABLE pour colonnes avant production
- [ ] FK déférées dans des migrations séparées avec `down()` qui supprime uniquement la FK
- [ ] Lookup Tables avec Seeder de données de référence
- [ ] Migrations idempotentes (IF NOT EXISTS)

**Agnosticisme**
- [ ] Aucune règle architecturale dans le code uniquement — déclarée en base si elle doit survivre à un changement de service

---

*Document maintenu par ATANAX Inc. — Montréal, Canada*
*Dernière mise à jour : Juin 2026*
*À lire conjointement avec : `AXIONS_KERNEL.md` et `ATANAX_ARCHITECTURE.md`*

# AXIONS KERNEL — Design Rules & Architecture Contract

> **Document officiel de référence pour tout développeur contribuant à l'écosystème Axions360.**
> Ces règles ne sont pas des recommandations. Ce sont des contrats d'architecture.
> Toute violation de ces règles crée une dette technique irréversible.
> À lire conjointement avec : `AXIONS_SCHEMA_RULES.md` et `ATANAX_ARCHITECTURE.md`

---

## Table des matières

1. [Philosophie fondamentale](#1-philosophie-fondamentale)
2. [Les entités noyau (Core Schema)](#2-les-entités-noyau-core-schema)
3. [Règles d'isolation multi-tenant](#3-règles-disolation-multi-tenant)
4. [Règles d'extension de modules](#4-règles-dextension-de-modules)
5. [Système de permissions (RBAC)](#5-système-de-permissions-rbac)
6. [Interface transactionnelle universelle](#6-interface-transactionnelle-universelle)
7. [Règles de versioning et migrations](#7-règles-de-versioning-et-migrations)
8. [Règles de nommage](#8-règles-de-nommage)
9. [Règles d'activation de modules](#9-règles-dactivation-de-modules)
10. [Ce qui est interdit](#10-ce-qui-est-interdit)
11. [Annexe — Checklist PR](#11-annexe--checklist-pr)

---

## 1. Philosophie fondamentale

### 1.1 Single Source of Truth (SSOT)

Il n'existe **qu'une seule représentation** de chaque entité dans le système.

Un employé est dans `employees`. Pas dans `restaurant_staff`. Pas dans `hotel_staff`. Pas dans `commerce_employees`. **Dans `employees`.**

Une transaction est dans `transactions`. Pas dans `restaurant_sales`, pas dans `hotel_revenues`. **Dans `transactions`.**

Une personne physique est dans `people`. Qu'elle soit employée, cliente ou fournisseur, son identité de base est centralisée.

### 1.2 Activation, jamais Migration

Le système est conçu pour que tout client puisse passer de :

```
Synchro Restaurant (2026)
```

à :

```
Synchro Restaurant + Axions RH + Axions Finance + Axions ERP (2029)
```

**sans exporter une seule ligne de données.**

Les données créées par Synchro Restaurant en 2026 sont lisibles nativement par Axions RH en 2029. Ce principe est non-négociable.

### 1.3 Architecture en couches

```
┌─────────────────────────────────────────────┐
│                  TENANT                     │  ← Unité souveraine
├─────────────────────────────────────────────┤
│              AXIONS KERNEL                  │  ← Core Schema partagé
├──────────────┬──────────────┬───────────────┤
│   Synchro    │   Axions ERP │     Dymmo     │  ← Modules métier
│  Restaurant  │   Finance    │               │
│   Hôtel      │   RH / CRM  │               │
│   Commerce   │   Projects   │               │
└──────────────┴──────────────┴───────────────┘
```

**La règle** : Les modules métier ne se parlent pas directement. Ils passent tous par le Kernel.

### 1.4 Agnosticisme Technologique

> **Toute règle architecturale qui doit survivre à un changement de framework vit dans la base de données, pas dans le code applicatif.**

Le code lit les règles. Il ne les définit pas.

Si une contrainte (audit obligatoire, verrouillage optimiste, soft delete) vit uniquement dans un `BaseModel` Laravel, elle disparaît dès qu'un service Go ou Spring Boot accède à la même table. C'est une fausse sécurité.

**Application** : la table `entity_registry` (section 2.17) déclare pour chaque entité son tier de criticité et ses protections. Tout service — Laravel, Go, Spring Boot, Rust — consulte ce registre au démarrage.

Voir `AXIONS_SCHEMA_RULES.md` section 1 pour la règle complète et les exemples par langage.

### 1.5 Zéro ENUM hardcodé

Aucun statut, type ou catégorie n'est codé en dur dans le schéma. Tout passe par des Lookup Tables avec i18n. Voir `AXIONS_SCHEMA_RULES.md` section 4.

---

## 2. Les entités noyau (Core Schema)

Ces tables constituent le **Axions Kernel**. Elles sont installées sur **tout tenant**, même ceux qui n'ont que Synchro Restaurant. Elles ne sont jamais modifiées par un module métier.

> Toutes les tables entités respectent l'**Entity Contract V1** défini dans `AXIONS_SCHEMA_RULES.md` section 2. Les protections (row_version, audit, soft_delete) sont déclarées dans `entity_registry` (section 2.17) — pas dans le code applicatif.

### Catalogue des tables Core

```
IDENTITÉ & ACCÈS
├── people                  ← identité humaine de base
├── users                   ← accès système (étend people)
├── employees               ← relation professionnelle (étend people)

STRUCTURE
├── tenants                 ← unité souveraine
├── organizations           ← entités légales du groupe
├── stores                  ← établissements physiques/virtuels
├── addresses               ← adresses réutilisables

RELATIONS EXTERNES
├── contacts                ← clients, fournisseurs, partenaires (étend people)

FINANCIER
├── transactions            ← interface universelle des flux financiers

DOCUMENTS
├── documents               ← factures, contrats, reçus

LANGUES & i18n
├── languages               ← catalogue global des langues
├── tenant_languages        ← langues activées par tenant
├── store_languages         ← langues activées par store

OPSCODES & OPERATION CODES
├── opscode_domains         ← domaines métier (AUTH, ORDER, HR...)
├── opscode_severities      ← niveaux de sévérité
├── opscodes                ← catalogue universel des événements système
├── opscode_translations    ← traductions des opscodes
├── operation_domains       ← domaines financiers (TRANSFER, PAYMENT...)
├── operation_impacts       ← impacts (DEBIT, CREDIT, NEUTRAL)
├── operation_codes         ← codes d'impact financier (Dymmo/Finance)
├── operation_code_i18n     ← traductions des operation codes

PERMISSIONS & RÔLES
├── roles                   ← rôles par tenant/store
├── permissions             ← catalogue des permissions
├── role_permissions        ← pivot
├── user_roles              ← attribution des rôles

TRAÇABILITÉ
├── audit_logs              ← historique immuable (append-only)
├── domain_events           ← événements domaine persistés

MODULES
├── module_activations      ← registre des modules activés par tenant

REGISTRE D'ARCHITECTURE
└── entity_registry         ← tier de criticité et protections de chaque entité (agnostique au framework)
```

---

### 2.1 `tenants`

L'unité souveraine. Tout appartient à un tenant. Exception G-ROOT-01 : seule table sans `tenant_id`.

```sql
tenants
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── name                VARCHAR(255) NOT NULL              -- "Groupe Diallo & Fils"
├── slug                VARCHAR(100) UNIQUE NOT NULL       -- "groupe-diallo"
├── country_code        CHAR(2) NOT NULL                   -- ISO 3166-1 : "GN", "SN", "CI"
├── currency_code       CHAR(3) NOT NULL                   -- ISO 4217 : "GNF", "XOF", "USD"
├── timezone            VARCHAR(50) NOT NULL               -- "Africa/Conakry"
├── default_locale      VARCHAR(10) NOT NULL DEFAULT 'fr'  -- langue par défaut
├── plan_code           VARCHAR(30) NOT NULL DEFAULT 'FREE' REFERENCES tenant_plans(code)
├── status_code         VARCHAR(30) NOT NULL DEFAULT 'ACTIVE' REFERENCES tenant_statuses(code)
├── metadata            JSONB NULL
├── created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
```

**Règle** : `tenant_id` est présent dans **toutes** les autres tables du système sans exception.

---

### 2.2 `people`

Identité humaine de base. Toute personne physique — employée, cliente, fournisseure — a une entrée ici.

```sql
people
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── first_name          VARCHAR(100) NOT NULL
├── last_name           VARCHAR(100) NOT NULL
├── preferred_name      VARCHAR(100) NULL               -- nom d'usage
├── date_of_birth       DATE NULL
├── gender_code         VARCHAR(20) NULL REFERENCES genders(code)
├── phone               VARCHAR(20) NULL
├── email               VARCHAR(255) NULL
├── address_id          BIGINT UNSIGNED NULL REFERENCES addresses(id)
├── nationality_code    CHAR(2) NULL                    -- ISO 3166-1
├── preferred_locale    VARCHAR(10) NULL REFERENCES languages(code)
├── metadata            JSONB NULL
├── row_version          INTEGER NOT NULL DEFAULT 0
├── created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP

INDEX idx_people_tenant (tenant_id)
```

**Règle** : `people` est la racine de toute identité humaine. `users`, `employees` et `contacts` étendent `people`, ils ne la remplacent pas.

---

### 2.3 `users`

Accès système. Un user **est** une personne (`people`) avec des credentials d'authentification.

```sql
users
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── person_id           BIGINT UNSIGNED NOT NULL REFERENCES people(id)
├── email               VARCHAR(255) NULL                -- NULL si auth par téléphone
├── phone               VARCHAR(20) NULL                 -- NULL si auth par email
├── phone_verified_at   TIMESTAMP WITH TIME ZONE NULL
├── email_verified_at   TIMESTAMP WITH TIME ZONE NULL
├── password            VARCHAR(255) NOT NULL
├── avatar_url          VARCHAR(500) NULL
├── last_login_at       TIMESTAMP WITH TIME ZONE NULL
├── status_code         VARCHAR(30) NOT NULL DEFAULT 'ACTIVE' REFERENCES user_statuses(code)
├── metadata            JSONB NULL
├── row_version          INTEGER NOT NULL DEFAULT 0
├── created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP

UNIQUE (tenant_id, email)
UNIQUE (tenant_id, phone)
INDEX idx_users_tenant (tenant_id)
INDEX idx_users_person (person_id)
```

**Règle** : Un user est lié à exactement un tenant. L'authentification inter-tenant n'existe pas.

---

### 2.4 `employees`

Relation professionnelle. Un employee **est** une personne (`people`), mais pas nécessairement un user.

```sql
employees
├── id                   BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                 CHAR(26) UNIQUE NOT NULL
├── tenant_id            BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── person_id            BIGINT UNSIGNED NOT NULL REFERENCES people(id)
├── user_id              BIGINT UNSIGNED NULL REFERENCES users(id)  -- NULL = pas d'accès système
├── store_id             BIGINT UNSIGNED NULL REFERENCES stores(id)
├── employee_number      VARCHAR(50) NULL
├── job_title            VARCHAR(255) NULL
├── department           VARCHAR(255) NULL
├── hire_date            DATE NULL
├── end_date             DATE NULL
├── employment_type_code VARCHAR(30) NOT NULL REFERENCES employment_types(code)
├── status_code          VARCHAR(30) NOT NULL DEFAULT 'ACTIVE' REFERENCES employee_statuses(code)
├── metadata             JSONB NULL
├── row_version          INTEGER NOT NULL DEFAULT 0
├── created_at           TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at           TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP

UNIQUE (tenant_id, person_id)
INDEX idx_employees_store (store_id)
INDEX idx_employees_person (person_id)
```

**Règle** : `user_id` est nullable — un journalier, un employé sans accès au logiciel, existe dans `employees` sans entrée dans `users`. Le module Axions RH crée `hr_employee_profiles` qui pointe vers `employees.id`.

---

### 2.5 `contacts`

Personnes ou entités externes (clients, fournisseurs, partenaires). Peut pointer vers `people` si c'est une personne physique.

```sql
contacts
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── person_id           BIGINT UNSIGNED NULL REFERENCES people(id)  -- si personne physique
├── type_code           VARCHAR(30) NOT NULL REFERENCES contact_types(code)
├── name                VARCHAR(255) NOT NULL               -- nom d'affichage ou raison sociale
├── email               VARCHAR(255) NULL
├── phone               VARCHAR(20) NULL
├── address_id          BIGINT UNSIGNED NULL REFERENCES addresses(id)
├── tax_id              VARCHAR(100) NULL
├── preferred_locale    VARCHAR(10) NULL REFERENCES languages(code)
├── preferred_currency  CHAR(3) NULL
├── notes               TEXT NULL
├── status_code         VARCHAR(30) NOT NULL DEFAULT 'ACTIVE' REFERENCES contact_statuses(code)
├── metadata            JSONB NULL
├── row_version          INTEGER NOT NULL DEFAULT 0
├── created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP

INDEX idx_contacts_tenant_type (tenant_id, type_code)
INDEX idx_contacts_person (person_id)
```

---

### 2.6 `organizations`

Les entités légales du groupe (sociétés, filiales, branches).

```sql
organizations
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── parent_id           BIGINT UNSIGNED NULL REFERENCES organizations(id)
├── name                VARCHAR(255) NOT NULL
├── legal_name          VARCHAR(255) NULL
├── registration_number VARCHAR(100) NULL               -- RCCM, NIF, etc.
├── tax_id              VARCHAR(100) NULL
├── country_code        CHAR(2) NOT NULL
├── currency_code       CHAR(3) NOT NULL
├── address_id          BIGINT UNSIGNED NULL REFERENCES addresses(id)
├── status_code         VARCHAR(30) NOT NULL DEFAULT 'ACTIVE' REFERENCES organization_statuses(code)
├── metadata            JSONB NULL
├── row_version          INTEGER NOT NULL DEFAULT 0
├── created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
```

---

### 2.7 `stores`

Tout point opérationnel physique ou virtuel (restaurant, hôtel, boutique, agence).

```sql
stores
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── organization_id     BIGINT UNSIGNED NULL REFERENCES organizations(id)
├── name                VARCHAR(255) NOT NULL
├── type_code           VARCHAR(30) NOT NULL REFERENCES store_types(code)
├── address_id          BIGINT UNSIGNED NULL REFERENCES addresses(id)
├── timezone            VARCHAR(50) NULL                -- peut différer du tenant
├── currency_code       CHAR(3) NULL                    -- peut différer du tenant
├── status_code         VARCHAR(30) NOT NULL DEFAULT 'ACTIVE' REFERENCES store_statuses(code)
├── metadata            JSONB NULL
├── row_version          INTEGER NOT NULL DEFAULT 0
├── created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP

INDEX idx_stores_tenant (tenant_id)
INDEX idx_stores_type (type_code)
```

---

### 2.8 `addresses`

Adresses réutilisables par tous les modules. Jamais dupliquées.

```sql
addresses
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── line1               VARCHAR(255) NOT NULL
├── line2               VARCHAR(255) NULL
├── city                VARCHAR(100) NOT NULL
├── state               VARCHAR(100) NULL
├── postal_code         VARCHAR(20) NULL
├── country_code        CHAR(2) NOT NULL
├── latitude            DECIMAL(10, 8) NULL
├── longitude           DECIMAL(11, 8) NULL
├── created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
```

---

### 2.9 `transactions`

**Interface universelle pour tout flux financier du système.**

Chaque vente, achat, paiement, remboursement crée une entrée ici. Le module Finance lit cette table directement sans connaître les tables internes des modules.

```sql
transactions
├── id                   BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                 CHAR(26) UNIQUE NOT NULL
├── tenant_id            BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── store_id             BIGINT UNSIGNED NULL REFERENCES stores(id)
├── amount               BIGINT NOT NULL                  -- JAMAIS DECIMAL — sous-unités
├── currency_code        CHAR(3) NOT NULL
├── exchange_rate        DECIMAL(18, 8) NULL
├── operation_code       VARCHAR(50) NOT NULL REFERENCES operation_codes(code)
├── source_opcode_code   VARCHAR(128) NOT NULL REFERENCES opscodes(code)
├── direction            CHAR(3) NOT NULL                 -- 'in' ou 'out'
├── status_code          VARCHAR(30) NOT NULL REFERENCES transaction_statuses(code)
├── contact_id           BIGINT UNSIGNED NULL REFERENCES contacts(id)
├── reference_id         CHAR(26) NULL                    -- ULID de l'entité source
├── reference            VARCHAR(255) NULL                -- numéro de reçu, facture
├── idempotency_key      VARCHAR(128) NOT NULL
├── metadata             JSONB NULL
├── occurred_at          TIMESTAMP WITH TIME ZONE NOT NULL
├── created_at           TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at           TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP

UNIQUE (tenant_id, idempotency_key)
INDEX idx_transactions_tenant_op (tenant_id, operation_code)
INDEX idx_transactions_store (store_id)
INDEX idx_transactions_occurred (occurred_at)
```

> **Règle critique** : Les montants sont **toujours** en BIGINT sous-unités. 1 000 GNF = `1000`. 10.50 USD = `1050`. **Jamais DECIMAL. Jamais FLOAT.**

---

### 2.10 `documents`

Factures, contrats, reçus — tout document produit par le système.

```sql
documents
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── store_id            BIGINT UNSIGNED NULL REFERENCES stores(id)
├── type_code           VARCHAR(30) NOT NULL REFERENCES document_types(code)
├── title               VARCHAR(255) NOT NULL
├── number              VARCHAR(100) NULL
├── language_code       VARCHAR(10) NULL REFERENCES languages(code)
├── file_path           VARCHAR(500) NULL
├── file_size           INT UNSIGNED NULL
├── mime_type           VARCHAR(100) NULL
├── source_opcode_code  VARCHAR(128) NULL REFERENCES opscodes(code)
├── reference_id        CHAR(26) NULL
├── contact_id          BIGINT UNSIGNED NULL REFERENCES contacts(id)
├── issued_at           DATE NULL
├── expires_at          DATE NULL
├── status_code         VARCHAR(30) NOT NULL DEFAULT 'DRAFT' REFERENCES document_statuses(code)
├── metadata            JSONB NULL
├── row_version          INTEGER NOT NULL DEFAULT 0
├── created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
```

---

### 2.11 `audit_logs`

Traçabilité complète et immuable. Conformité OHADA. **Append-only — zéro UPDATE, zéro DELETE.**

```sql
audit_logs
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── user_id             BIGINT UNSIGNED NULL REFERENCES users(id)
├── store_id            BIGINT UNSIGNED NULL REFERENCES stores(id)
├── opscode_code        VARCHAR(128) NOT NULL REFERENCES opscodes(code)
├── entity_type         VARCHAR(100) NOT NULL               -- "Order", "Employee"
├── entity_id           BIGINT UNSIGNED NOT NULL
├── old_values          JSONB NULL
├── new_values          JSONB NULL
├── ip_address          VARCHAR(45) NULL
├── user_agent          VARCHAR(500) NULL
├── occurred_at         TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP

-- Pas de updated_at — cette table ne se modifie jamais
-- Pas de version — append-only par définition
INDEX idx_audit_tenant_entity (tenant_id, entity_type, entity_id)
INDEX idx_audit_opscode (opscode_code)
INDEX idx_audit_user (user_id)
```

---

### 2.12 `domain_events`

Événements domaine persistés pour rejouabilité et communication inter-modules.

```sql
domain_events
├── id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid                CHAR(26) UNIQUE NOT NULL
├── tenant_id           BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── opscode_code        VARCHAR(128) NOT NULL REFERENCES opscodes(code)
├── aggregate_type      VARCHAR(100) NOT NULL               -- "Order", "Employee"
├── aggregate_id        BIGINT UNSIGNED NOT NULL
├── payload             JSONB NOT NULL
├── occurred_at         TIMESTAMP WITH TIME ZONE NOT NULL
├── processed_at        TIMESTAMP WITH TIME ZONE NULL

INDEX idx_domain_events_aggregate (aggregate_type, aggregate_id)
INDEX idx_domain_events_tenant_opscode (tenant_id, opscode_code)
```

---

### 2.13 `languages`, `tenant_languages`, `store_languages`

Voir `AXIONS_SCHEMA_RULES.md` section 4 pour le schéma complet.

```sql
-- Résumé
languages          ← catalogue global (pas de tenant_id)
tenant_languages   ← langues activées par tenant avec langue par défaut
store_languages    ← langues activées par store (sous-ensemble du tenant)
```

---

### 2.14 `opscodes` et tables associées

Voir `AXIONS_SCHEMA_RULES.md` section 5 pour le schéma complet.

```sql
-- Résumé
opscode_domains        ← domaines métier (AUTH, ORDER, HR, SYNC...)
opscode_severities     ← niveaux (INFO, WARNING, ERROR, CRITICAL...)
opscodes               ← catalogue des événements système
opscode_translations   ← traductions multilingues des opscodes
```

---

### 2.15 `operation_codes` et tables associées

Voir `AXIONS_SCHEMA_RULES.md` section 6 pour le schéma complet.

```sql
-- Résumé
operation_domains    ← domaines financiers (TRANSFER, PAYMENT, FEE, FX...)
operation_impacts    ← impacts (DEBIT, CREDIT, NEUTRAL)
operation_codes      ← codes d'impact financier (Dymmo/Finance)
operation_code_i18n  ← traductions
```

---

### 2.16 `entity_registry`

**Table de référence architecturale.** Déclare le tier de criticité et les protections de chaque entité du système. Consultée au démarrage par tout service — Laravel, Go, Spring Boot, Rust — pour appliquer les règles correctes sans dépendre d'un framework spécifique.

Voir `AXIONS_SCHEMA_RULES.md` sections 1 et 3 pour la définition complète et les exemples par langage.

```sql
entity_registry
├── entity_name       VARCHAR(100) PRIMARY KEY   -- 'employees', 'transactions'
├── module_slug       VARCHAR(50)  NOT NULL       -- 'axions-kernel', 'synchro-restaurant'
├── contract_version  SMALLINT     NOT NULL DEFAULT 1
├── tier              SMALLINT     NOT NULL       -- 1=CRITICAL 2=STANDARD 3=OPERATIONAL 4=REFERENCE
├── has_row_version   BOOLEAN      NOT NULL DEFAULT false
├── has_audit         BOOLEAN      NOT NULL DEFAULT false
├── has_soft_delete   BOOLEAN      NOT NULL DEFAULT false
├── has_idempotency   BOOLEAN      NOT NULL DEFAULT false
├── has_pii           BOOLEAN      NOT NULL DEFAULT false
├── description       TEXT         NULL
├── created_at        TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
└── updated_at        TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
```

**Règle** : Toute nouvelle table entité doit avoir une entrée dans `entity_registry` avant d'être mise en production. Cette entrée est créée par le Seeder du module lors de son activation.

---

### 2.17 `module_activations`

Registre de tous les modules activés par tenant.

```sql
module_activations
├── id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── tenant_id       BIGINT UNSIGNED NOT NULL REFERENCES tenants(id)
├── module_slug     VARCHAR(100) NOT NULL
├── version         VARCHAR(20) NOT NULL
├── status_code     VARCHAR(30) NOT NULL REFERENCES module_statuses(code)
├── activated_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
├── activated_by    BIGINT UNSIGNED NULL REFERENCES users(id)
├── config          JSONB NULL

UNIQUE (tenant_id, module_slug)
```

---

## 3. Règles d'isolation multi-tenant

### 3.1 La règle absolue

**Toute table du système doit avoir un `tenant_id`.** Sans exception. Même les tables de configuration. Même les tables de cache.

**Exception unique documentée (G-ROOT-01)** : les tables lookup globales (statiques, partagées par tous les tenants) n'ont pas de `tenant_id` — elles contiennent des données système, pas des données client.

### 3.2 Scope automatique

Chaque requête est scopée au tenant actif via le BaseModel, jamais manuellement dans chaque controller.

```php
// BaseModel.php
protected static function booted(): void
{
    static::addGlobalScope('tenant', function (Builder $builder) {
        if ($tenantId = app('currentTenant')?->id) {
            $builder->where(static::getTable() . '.tenant_id', $tenantId);
        }
    });
}
```

### 3.3 Isolation store-à-store

Un utilisateur du Store A (Restaurant) ne peut pas voir les données du Store B (Hôtel) même s'ils appartiennent au même tenant, sauf si son rôle le permet explicitement.

Le modèle de permissions est `Tenant → Organization → Store → Role → Permission`, et non `Tenant → Role → Permission`.

---

## 4. Règles d'extension de modules

### 4.1 La règle d'extension

**Un module ne modifie jamais une table core. Il crée ses propres tables d'extension.**

```php
// ❌ INTERDIT — modifier une table core
Schema::table('employees', function (Blueprint $table) {
    $table->string('matricule_rh');  // NON
});

// ✅ CORRECT — créer une table d'extension
Schema::create('hr_employee_profiles', function (Blueprint $table) {
    $table->id();
    $table->char('ulid', 26)->unique();
    $table->unsignedBigInteger('tenant_id');
    $table->unsignedBigInteger('employee_id');
    $table->string('matricule')->nullable();
    $table->date('hire_date')->nullable();
    $table->integer('row_version')->default(0);
    $table->timestamps();

    $table->foreign('tenant_id')->references('id')->on('tenants');
    $table->foreign('employee_id')->references('id')->on('employees');
    $table->unique(['employee_id']);
});
```

### 4.2 Convention de nommage des tables d'extension

```
{module}_{entite_core}_profiles   → extensions 1-à-1
{module}_{concept_propre}         → entités propres au module
```

Exemples :
- `hr_employee_profiles` — extension RH de employees
- `restaurant_menu_items` — entité propre à Synchro Restaurant
- `hotel_room_types` — entité propre à Synchro Hôtel

### 4.3 Toute entité de module doit référencer tenant_id et respecter Entity Contract V1

Voir `AXIONS_SCHEMA_RULES.md` section 1 — Entity Contract V1.

### 4.4 Les transactions métier passent par `transactions`

Quand Synchro Restaurant enregistre une vente, il crée :
1. Une entrée dans `restaurant_orders` (sa propre table)
2. **ET** une entrée dans `transactions` (table core)

```php
DB::transaction(function () use ($order) {
    $order->save();

    Transaction::create([
        'tenant_id'          => $order->tenant_id,
        'store_id'           => $order->store_id,
        'amount'             => $order->total_amount,     // BIGINT sous-unités
        'currency_code'      => $order->currency_code,
        'operation_code'     => 'MERCHANT_PAYMENT_CREDIT',
        'source_opcode_code' => 'synchro.restaurant.order.paid',
        'direction'          => 'in',
        'status_code'        => 'COMPLETED',
        'idempotency_key'    => $order->ulid . '_sale',
        'occurred_at'        => $order->completed_at,
    ]);
});
```

---

## 5. Système de permissions (RBAC)

### 5.1 Structure

```sql
roles
├── id, ulid, tenant_id
├── store_id      NULL  -- NULL = rôle global tenant, non-NULL = rôle store-specific
├── name          -- "Caissier", "Manager Restaurant"
├── slug          -- "cashier", "restaurant_manager"
├── is_system     BOOLEAN  -- rôles système non-modifiables
├── status_code   REFERENCES role_statuses(code)
├── row_version, timestamps

permissions
├── id, name, module, description, timestamps
-- name : "synchro-restaurant.orders.create"

role_permissions
├── role_id, permission_id, timestamps

user_roles
├── id, user_id, role_id
├── store_id      NULL  -- NULL = rôle s'applique à tout le tenant
├── granted_at, granted_by, expires_at, timestamps
```

### 5.2 Convention de nommage des permissions

```
{module}.{entite}.{action}

Exemples :
synchro-restaurant.orders.create
synchro-restaurant.orders.delete
axions-hr.employees.view
axions-hr.payroll.process
axions-finance.reports.export
axions-erp.tenants.manage
```

### 5.3 Vérification dans le code

```php
Gate::authorize('synchro-restaurant.orders.create');
$user->can('axions-hr.employees.view');
$user->canInStore($store, 'synchro-restaurant.orders.delete');
```

---

## 6. Interface transactionnelle universelle

### 6.1 Règle absolue sur les montants monétaires

**Tout montant monétaire est stocké en BIGINT représentant les sous-unités.**

| Devise | Sous-unité | 1 000 GNF | 10.50 USD | 5 000 XOF |
|--------|-----------|-----------|-----------|-----------|
| GNF    | 1 GNF     | `1000`    | —         | —         |
| USD    | 1 cent    | —         | `1050`    | —         |
| XOF    | 1 FCFA    | —         | —         | `5000`    |

**Jamais DECIMAL. Jamais FLOAT. Jamais VARCHAR.**

### 6.2 Lien opscode ↔ operation_code

```
opscode            = ce qui s'est passé (événement métier)
operation_code     = l'impact financier sur le grand livre

synchro.restaurant.order.paid  →  MERCHANT_PAYMENT_CREDIT
                                →  MERCHANT_FEE  (commission Dymmo)
```

---

## 7. Règles de versioning et migrations

### 7.1 Les migrations core sont immuables

Une fois qu'une migration du Kernel est exécutée en production, elle ne se modifie **jamais**.

```
✅ 2026_01_15_000001_create_tenants_table.php       ← ne jamais toucher
✅ 2026_03_20_000001_add_locale_to_tenants.php      ← nouvelle migration si ajout nécessaire
❌ Modifier 2026_01_15_000001_create_tenants_table.php  ← INTERDIT en production
```

### 7.2 Ordre strict des migrations

```
1. axions-kernel-lookup   (lookup tables globales + i18n)
2. axions-kernel-core     (tables core : people, users, employees, tenants...)
3. axions-kernel-i18n     (languages, tenant_languages, store_languages)
4. axions-kernel-opscodes (opscodes, operation_codes et leurs tables)
5. axions-auth            (roles, permissions)
6. synchro-*              (modules Synchro)
7. axions-hr, axions-finance, axions-crm...
8. dymmo
9. integration-layer
```

Les dépendances vont toujours **vers le bas** (vers le kernel). Un module ne peut jamais créer une FK vers une table d'un module de même niveau ou supérieur.

### 7.3 Colonne `metadata JSONB`

Chaque table core a une colonne `metadata JSONB NULL` pour les données non-structurées temporaires.

**Règle** : Si une donnée `metadata` est utilisée par 3 modules différents ou plus, elle doit être promue en colonne propre via une nouvelle migration.

Voir `AXIONS_SCHEMA_RULES.md` section 9 pour les règles complètes de migrations.

---

## 8. Règles de nommage

### 8.1 Tables

| Type | Convention | Exemple |
|------|-----------|---------|
| Tables core | `{entite_pluriel}` | `employees`, `transactions` |
| Lookup tables | `{entite}_statuses`, `{entite}_types` | `order_statuses`, `store_types` |
| Tables i18n lookup | `{lookup_table}_i18n` | `order_status_i18n` |
| Tables module | `{module}_{entite}` | `restaurant_orders`, `hotel_rooms` |
| Tables extension | `{module}_{entite_core}_profiles` | `hr_employee_profiles` |
| Tables translations | `{entite}_translations` | `menu_item_translations` |
| Tables pivot | `{table_a}_{table_b}` (ordre alpha) | `role_permissions`, `user_roles` |

### 8.2 Colonnes

- Noms en `snake_case`
- Foreign keys : `{table_singulier}_id` → `tenant_id`, `store_id`, `person_id`
- Références lookup : `{contexte}_code` → `status_code`, `type_code`, `employment_type_code`
- Timestamps : `created_at`, `updated_at`, `deleted_at`
- Dates métier : `{contexte}_at` → `hire_date`, `occurred_at`, `processed_at`
- Booléens : `is_{état}` ou `has_{feature}` → `is_auditable`, `has_cloud_sync`
- Codes opscodes : `opscode_code` → référence vers `opscodes.code`

### 8.3 Identifiants

- `id` : BIGINT interne, usage SQL uniquement, **jamais exposé en dehors du système**
- `ulid` : CHAR(26), usage externe, URLs, API responses

```php
// ✅ Correct
GET /api/stores/01HXYZ123ABC456DEF789GHI00   ← ULID dans l'URL

// ❌ Interdit
GET /api/stores/42   ← ID entier exposé
```

---

## 9. Règles d'activation de modules

### 9.1 Chaque module est activable indépendamment

```bash
php artisan axions:module:activate synchro-restaurant --tenant=01HXYZ
php artisan axions:module:activate axions-hr --tenant=01HXYZ
```

Cette commande :
1. Vérifie les dépendances déclarées du module
2. Exécute les migrations propres au module
3. Seed les opscodes du module dans le catalogue
4. Seed les permissions du module
5. Seed les rôles système du module
6. Enregistre dans `module_activations`

### 9.2 Dépendances déclarées

```php
// modules/AxionsHR/ModuleServiceProvider.php
public array $dependencies = [
    'axions-kernel',
    'axions-auth',
    // Pas de dépendance vers synchro-restaurant — HR est indépendant
];
```

### 9.3 Ce que l'activation ne fait PAS

- Elle ne supprime pas de données existantes
- Elle ne modifie pas les tables core
- Elle ne recrée pas des tables déjà existantes
- Elle ne touche pas aux modules déjà activés

---

## 10. Ce qui est interdit

| ❌ Interdit | ✅ Correct |
|-------------|-----------|
| `ENUM` hardcodé dans le schéma | Lookup Table avec `_code` + i18n sidecar |
| `ALTER TABLE employees ADD COLUMN` dans un module | Créer `hr_employee_profiles` |
| Créer `restaurant_staff` à la place de `employees` | `employees` + extension module |
| `user_id NOT NULL` sur `employees` | `user_id NULL` — un employé peut ne pas avoir d'accès |
| DECIMAL ou FLOAT pour les montants | BIGINT sous-unités |
| Exposer `id` entier en API ou URL | Utiliser `ulid` |
| Requête sans scope `tenant_id` | Scope automatique via BaseModel |
| Module qui accède aux tables d'un autre module | Events ou tables Core uniquement |
| `UPDATE` ou `DELETE` sur `audit_logs` ou `domain_events` | Append-only — interdit absolument |
| Modifier une migration core en production | Créer une nouvelle migration |
| `SELECT *` | Sélectionner uniquement les colonnes nécessaires |
| Opération financière sans `idempotency_key` | Contrainte UNIQUE sur idempotency_key |
| Texte affiché hardcodé dans le code | Opscode translations ou i18n |
| Données PII en clair en base | Chiffrement AES-256-GCM + blind index |
| Clé cryptographique stockée en base | `key_ref` KMS uniquement |

---

## 11. Annexe — Checklist PR

- [ ] Entity Contract V1 respectée (ulid, tenant_id, status_code via lookup, version, timestamps WITH TIME ZONE)
- [ ] Aucun ENUM hardcodé — Lookup Table avec i18n sidecar
- [ ] Contenu multilingue visible : table `_translations` ou `_i18n`
- [ ] Langues référencées via `languages(code)`, jamais des strings libres
- [ ] Nouveaux événements métier : opscode déclaré + traductions FR/EN minimum
- [ ] Nouvelles opérations financières : operation_code dans le catalogue
- [ ] Montants en BIGINT sous-unités
- [ ] Entités exposées en API ont un `ulid`
- [ ] Aucune table core modifiée
- [ ] Extensions de tables core dans des tables `{module}_{entity}_profiles`
- [ ] Transactions financières créent une entrée dans `transactions` avec `operation_code` + `source_opcode_code`
- [ ] Actions sensibles auditées dans `audit_logs` avec `opscode_code`
- [ ] Opérations monétaires ont une `idempotency_key` avec contrainte UNIQUE
- [ ] Données PII chiffrées avec `blind_index` et `key_version_id`
- [ ] Une migration = une table principale
- [ ] Zéro ALTER TABLE pour colonnes avant production
- [ ] FK déférées dans des migrations séparées
- [ ] Lookup Tables ont un Seeder

---

*Document maintenu par ATANAX Inc. — Montréal, Canada*
*Dernière mise à jour : Juin 2026*
*Ce document évolue uniquement par décision architecturale explicite du Founder.*
*À lire conjointement avec : `AXIONS_SCHEMA_RULES.md` et `ATANAX_ARCHITECTURE.md`*

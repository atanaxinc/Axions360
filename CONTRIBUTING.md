# CONTRIBUTING — Engineering Rules & Git Strategy

> **Ce document est le contrat d'ingénierie d'Axions360.**
> Il définit la stratégie de branches, le versioning des contrats, les lois de mutation de données, et les règles de contribution.
> Toute Pull Request qui viole ces règles est rejetée sans exception.
> À lire conjointement avec : `ATANAX_ARCHITECTURE.md`, `AXIONS_KERNEL.md`, `AXIONS_SCHEMA_RULES.md`

---

## Table des matières

1. [Pourquoi ces règles existent](#1-pourquoi-ces-règles-existent)
2. [Topographie Git — Branches](#2-topographie-git--branches)
3. [La Matrice des Contrats ATANAX](#3-la-matrice-des-contrats-atanax)
4. [Les Trois Lois de Mutation de Données](#4-les-trois-lois-de-mutation-de-données)
5. [SemVer — Versioning applicatif](#5-semver--versioning-applicatif)
6. [Règles de Pull Request](#6-règles-de-pull-request)
7. [Règles de Migration SQL](#7-règles-de-migration-sql)
8. [Ce qui est interdit](#8-ce-qui-est-interdit)

---

## 1. Pourquoi ces règles existent

Axions360 a une contrainte que la plupart des projets open source n'ont pas : **le code et les données vivent en deux endroits simultanément**.

```
Cloud (app.axions360.com)    →  branche main, déploiement continu
Terrain local (Synchro)      →  version figée installée chez le client
```

Un restaurant à Conakry peut tourner sur Synchro v1.2 pendant que le Cloud est déjà à v2.0. Si une migration sur `main` casse la compatibilité des données, ce restaurant perd ses données lors de la prochaine synchronisation.

**Le code peut évoluer vite. La base de données doit évoluer prudemment.**

C'est cette contrainte qui dicte toutes les règles qui suivent.

---

## 2. Topographie Git — Branches

### 2.1 Structure des branches

```
main  (tronc vivant — Cloud SaaS)
  │
  │  Nouvelles fonctionnalités modules
  │  Migrations STRICTEMENT additives
  │  Tests non-régression Sync Engine obligatoires
  │
  ├───> release/v1.x  (Synchro Local terrain — v1)
  │         │
  │         │  Bugfixes uniquement
  │         │  Security patches uniquement
  │         │  ZÉRO modification de schéma SQL
  │         │
  │         └── tags: v1.0.0, v1.0.1, v1.1.0 ...
  │
  └───> release/v2.x  (Synchro Local terrain — v2)
            │
            │  Bugfixes + security patches
            │  Backport depuis main si validé
            │
            └── tags: v2.0.0, v2.0.1 ...
```

### 2.2 Règles par branche

#### `main` — Tronc vivant

- Reçoit toutes les nouvelles fonctionnalités
- Déployé en continu sur `app.axions360.com`
- Toute migration SQL doit être **strictement additive** (voir section 4)
- Toute PR fusionnée sur `main` doit passer la suite de tests Sync Engine
- Jamais de tag de version sur `main` — les tags vivent sur les branches `release/`

#### `release/vX.x` — Branches de maintenance terrain

- Créée au moment de la sortie d'une version stable Synchro Local
- Reçoit **uniquement** des bugfixes et security patches
- **INTERDICTION ABSOLUE** de modifier le schéma SQL existant
- **INTERDICTION ABSOLUE** d'ajouter de nouvelles fonctionnalités
- Les correctifs sont d'abord appliqués sur `main`, puis **backportés** vers `release/vX.x`
- Chaque correctif produit un tag SemVer patch (ex: `v1.0.1`)

#### Tags SemVer

```
v1.0.0  →  sortie majeure Synchro Local v1
v1.0.1  →  bugfix / security patch
v1.1.0  →  fonctionnalité mineure backward-compatible
v2.0.0  →  nouvelle génération — peut briser la compat v1
```

### 2.3 Workflow d'un correctif terrain

```
1. Bug détecté sur Synchro Local v1 en Guinée
2. Fix développé et testé sur main
3. PR mergée sur main
4. Backport du fix sur release/v1.x (cherry-pick)
5. Tests de non-régression sur release/v1.x
6. Tag v1.0.X créé sur release/v1.x
7. Package de mise à jour distribué aux serveurs terrain
```

---

## 3. La Matrice des Contrats ATANAX

Chaque déploiement sur `main` doit valider cette matrice de compatibilité. Un service local Synchro peut tourner une version inférieure au Cloud — cette matrice garantit que la synchronisation reste possible.

| Vecteur | Identifiant | Règle d'évolution |
|---------|-------------|-------------------|
| **Code applicatif** | SemVer (ex: `v1.2.4`) | Isolé dans les branches `release/vX.x` |
| **Entity Contract** | `V_X` (ex: `Entity Contract V1`) | Déclaré dans `entity_registry` en base — toute table indique sa version |
| **Database Schema** | Migration timestamp | Rétrocompatible. Pas de DROP, pas de RENAME sur colonnes actives |
| **Sync Protocol** | Protocol Version (ex: `sync-v1`) | Géré par le Sync Engine. Si le payload évolue sur `main`, le décodeur `v1` reste actif |
| **Public API** | API Version (ex: `/api/v1/`) | Les routes de la version précédente ne sont jamais supprimées sans cycle de dépréciation |
| **Module** | Module SemVer (ex: `synchro-restaurant@1.4.0`) | Déclaré dans `module_activations` |

### 3.1 Déclaration de version dans le code

Chaque module déclare ses versions de contrats dans son `ModuleServiceProvider` :

```php
class SynchroRestaurantServiceProvider extends ModuleServiceProvider
{
    public string $moduleSlug    = 'synchro-restaurant';
    public string $moduleVersion = '1.4.0';
    public int    $entityContractVersion = 1;
    public string $syncProtocolVersion  = 'sync-v1';
    public string $apiVersion           = 'v1';

    public array $dependencies = [
        'axions-kernel',
        'axions-auth',
    ];
}
```

### 3.2 Handshake Sync Engine

Lors de la synchronisation, le client local déclare ses versions de contrats. Le Sync Engine Cloud les valide avant d'accepter ou de transmettre des deltas :

```
Client local  →  { entity_contract: "V1", sync_protocol: "sync-v1", module: "synchro-restaurant@1.2.0" }
Sync Engine   →  valide la compatibilité
              →  sérialise les deltas selon sync-v1 si le Cloud est en sync-v2
              →  refuse si entity_contract est incompatible
```

---

## 4. Les Trois Lois de Mutation de Données

Ces lois s'appliquent à **toute Pull Request** qui touche à une migration SQL ou à la structure d'une entité. Elles protègent les serveurs Synchro Local terrain contre les ruptures de compatibilité.

---

### Loi 1 — Additivité Exclusive

> **Si un module a besoin d'une nouvelle information, la migration crée une nouvelle colonne nullable ou avec un DEFAULT strict. L'ancienne colonne reste lisible par les anciennes versions locales de Synchro.**

```sql
-- ✅ CORRECT — additive, backward-compatible
ALTER TABLE restaurant_orders ADD COLUMN IF NOT EXISTS delivery_zone VARCHAR(100) NULL;

-- ❌ INTERDIT — destructive
ALTER TABLE restaurant_orders DROP COLUMN zone;
ALTER TABLE restaurant_orders RENAME COLUMN zone TO delivery_zone;
```

**Règle de nommage pour les colonnes de remplacement :**
```
Si `zone` doit devenir `delivery_zone` :
1. Créer `delivery_zone` nullable
2. Migrer les données dans l'application (pas en SQL pur)
3. Déprécier `zone` (ne plus l'écrire, continuer à le lire)
4. Supprimer `zone` uniquement après extinction de release/v1.x
```

---

### Loi 2 — Cycle de Dépréciation

> **Une colonne ou une route API utilisée par `release/v1.x` ne peut pas être supprimée sur `main` avant que l'ensemble du parc de serveurs locaux terrain n'ait été migré et validé en `release/v2.x`.**

```
Étape 1 : Colonne marquée @deprecated dans le code (mais conservée en base)
Étape 2 : Notification aux opérateurs terrain de migrer vers v2
Étape 3 : Vérification que tous les serveurs terrain ont migré (via telemetry)
Étape 4 : Suppression autorisée uniquement après confirmation complète
```

Aucune suppression de colonne ni de route ne peut être effectuée sans une issue GitHub documentant :
- La raison de la suppression
- La liste des branches `release/` affectées
- La date prévue d'extinction de la colonne/route
- La confirmation que tous les clients terrain ont migré

---

### Loi 3 — Isolation du Payload Sync

> **Le Sync Engine doit être capable de sérialiser et désérialiser les deltas de données selon la version du protocole déclarée par le client local lors de son handshake initial.**

Si le payload d'un événement change entre `sync-v1` et `sync-v2` :

```go
// ✅ CORRECT — le Sync Engine gère les deux versions
func (e *SyncEngine) SerializeDelta(delta Delta, protocol string) []byte {
    switch protocol {
    case "sync-v1":
        return e.serializeV1(delta)  // ancien format, maintenu
    case "sync-v2":
        return e.serializeV2(delta)  // nouveau format
    }
}

// ❌ INTERDIT — supprimer sync-v1 tant que des clients terrain l'utilisent
```

**La version du protocole Sync ne peut être dépréciée qu'après extinction complète de la branche `release/` correspondante.**

---

## 5. SemVer — Versioning applicatif

Axions360 suit le [Semantic Versioning 2.0.0](https://semver.org/).

```
MAJOR.MINOR.PATCH

MAJOR  →  rupture de compatibilité (nouvelle Entity Contract version, nouveau Sync Protocol)
MINOR  →  fonctionnalité backward-compatible (nouveau module, nouvelle API route)
PATCH  →  bugfix, security patch, correction de comportement
```

### 5.1 Règle MAJOR

Un incrément MAJOR signifie qu'une nouvelle branche `release/vX.x` est créée. La branche précédente entre en mode **maintenance uniquement** (patches et security seulement).

Un incrément MAJOR nécessite :
- Une nouvelle version du Sync Protocol si le payload a changé
- Une mise à jour de `entity_registry.contract_version` si l'Entity Contract a évolué
- Un plan de migration documenté pour les clients terrain

### 5.2 Règle MINOR

Un incrément MINOR sur `main` doit être backward-compatible avec toutes les branches `release/` actives. Les migrations SQL associées sont strictement additives (Loi 1).

### 5.3 Règle PATCH

Un incrément PATCH ne modifie **jamais** le schéma SQL. Uniquement des corrections de comportement, de sécurité, ou de performance.

---

## 6. Règles de Pull Request

### 6.1 Avant d'ouvrir une PR

1. Lire `ATANAX_ARCHITECTURE.md`, `AXIONS_KERNEL.md`, `AXIONS_SCHEMA_RULES.md`
2. Ouvrir une **Issue** décrivant le problème ou la fonctionnalité avant toute implémentation
3. Attendre une validation de l'équipe core avant de coder (évite les PRs rejetées)

### 6.2 Structure d'une PR

```
Titre    : [module] Description courte (50 caractères max)
Exemples :
  [kernel] Add entity_registry seeder for HR module
  [synchro-restaurant] Fix KDS connection timeout
  [axions-finance] Add SYSCOHADA journal type lookup

Corps :
  - Problème résolu / fonctionnalité ajoutée
  - Migrations SQL incluses (oui/non)
  - Compatibilité release/vX.x vérifiée (oui/non/N.A.)
  - Tests ajoutés (oui/non)
  - Breaking change (oui/non — si oui, PR rejetée sans plan de migration)
```

### 6.3 Checklist PR obligatoire

Toute PR doit cocher ces points avant review :

**Schema & Data**
- [ ] Entity Contract V1 respecté (id, ulid, tenant_id, status_code via Lookup, row_version si Tier 1/2, timestamps WITH TIME ZONE)
- [ ] Entrée créée dans `entity_registry` pour toute nouvelle table entité
- [ ] Aucun ENUM hardcodé — Lookup Table + i18n sidecar
- [ ] Migrations strictement additives (Loi 1)
- [ ] Aucune colonne supprimée ou renommée sans cycle de dépréciation (Loi 2)
- [ ] Montants en BIGINT sous-unités — jamais DECIMAL ou FLOAT
- [ ] Entités exposées en API ont un ULID — jamais l'id entier

**Architecture**
- [ ] Règle d'agnosticisme respectée — toute règle architecturale en base, pas dans le code uniquement
- [ ] Aucun module n'accède directement aux tables d'un autre module
- [ ] Communication inter-modules via Events (opscodes) uniquement
- [ ] Transactions financières → entrée dans `transactions` avec `operation_code` + `source_opscode_code`
- [ ] Actions sensibles → `audit_logs` avec `opscode_code`
- [ ] `audit_logs` et `domain_events` en append-only — zéro UPDATE, zéro DELETE

**Versioning**
- [ ] Sync Protocol non modifié, ou nouveau décodeur de version ajouté (Loi 3)
- [ ] `module_activations` mis à jour si le numéro de version du module change
- [ ] Compatibilité avec les branches `release/` actives vérifiée

**Tests**
- [ ] Tests unitaires sur les Services — coverage ≥ 90%
- [ ] Tests feature sur les endpoints API — coverage ≥ 80%
- [ ] Tests de non-régression Sync Engine passés

---

## 7. Règles de Migration SQL

Ces règles complètent `AXIONS_SCHEMA_RULES.md` section 11 avec les contraintes spécifiques au versioning terrain.

### 7.1 Migrations autorisées sur `main`

```sql
-- ✅ Ajouter une colonne nullable
ALTER TABLE employees ADD COLUMN IF NOT EXISTS linkedin_url VARCHAR(500) NULL;

-- ✅ Ajouter une table
CREATE TABLE IF NOT EXISTS hr_employee_profiles (...);

-- ✅ Ajouter un index
CREATE INDEX IF NOT EXISTS idx_orders_tenant_status ON restaurant_orders(tenant_id, status_code);

-- ✅ Ajouter une FK déférée (sur colonne déjà existante)
ALTER TABLE government_ids ADD CONSTRAINT fk_key_version
    FOREIGN KEY (key_version_id) REFERENCES kyc_key_versions(id);

-- ✅ Ajouter une valeur dans une Lookup Table
INSERT INTO order_statuses (code) VALUES ('DISPUTED') ON CONFLICT DO NOTHING;
```

### 7.2 Migrations interdites sans cycle de dépréciation

```sql
-- ❌ INTERDIT — supprimer une colonne active
ALTER TABLE employees DROP COLUMN department;

-- ❌ INTERDIT — renommer une colonne
ALTER TABLE employees RENAME COLUMN department TO team;

-- ❌ INTERDIT — changer le type d'une colonne
ALTER TABLE transactions ALTER COLUMN amount TYPE DECIMAL;

-- ❌ INTERDIT — supprimer une table
DROP TABLE restaurant_staff;

-- ❌ INTERDIT — ajouter une contrainte NOT NULL sans DEFAULT sur colonne existante
ALTER TABLE orders ALTER COLUMN notes SET NOT NULL;  -- casse les anciennes versions
```

### 7.3 Migration sur `release/vX.x` — Règle absolue

```
Sur une branche release/vX.x :
✅ Autorisé  : corrections de données incorrectes (UPDATE ciblé sur bug connu)
✅ Autorisé  : ajout d'index pour performance
❌ INTERDIT  : toute modification de structure (ADD COLUMN, DROP, RENAME, ALTER TYPE)
❌ INTERDIT  : toute nouvelle table
❌ INTERDIT  : toute suppression de table ou colonne
```

---

## 8. Ce qui est interdit

| ❌ Interdit | Raison |
|-------------|--------|
| Merger directement sur `release/vX.x` sans passer par `main` d'abord | Assure que le fix est dans le tronc principal |
| Supprimer une branche `release/` active sans extinction du parc terrain | Des serveurs locaux la référencent encore |
| Créer une migration destructive sur `main` | Viole la Loi 1 — rupture de compatibilité terrain |
| Modifier le Sync Protocol sans maintenir le décodeur de version précédente | Viole la Loi 3 — les clients terrain se déconnectent |
| Merger une PR sans l'Issue correspondante | Pas de traçabilité |
| Merger sans avoir passé la checklist PR | Risque de régression terrain |
| Tag de version sur `main` | Les tags vivent sur les branches `release/` |
| Force push sur `main` ou `release/vX.x` | Protections de branche activées — interdit |
| Supprimer une route API `/api/v1/` sans cycle de dépréciation | Des clients terrain l'utilisent encore |

---

*Document maintenu par ATANAX Inc. — Montréal, Canada*
*Dernière mise à jour : Juin 2026*
*Ce document évolue uniquement par décision architecturale explicite du Founder.*

# ATANAX — Architecture Master Document

> **Document de référence absolu pour tout l'écosystème ATANAX.**
> Toute décision technique doit être cohérente avec ce document.
> Toute déviation doit faire l'objet d'une décision architecturale explicite du Founder.

---

## Table des matières

1. [Philosophie architecturale](#1-philosophie-architecturale)
2. [Vue d'ensemble de l'écosystème](#2-vue-densemble-de-lécosystème)
3. [Stack technologique par couche](#3-stack-technologique-par-couche)
4. [Architecture du Monolithe Modulaire](#4-architecture-du-monolithe-modulaire)
5. [Services externes critiques](#5-services-externes-critiques)
6. [Integration Layer](#6-integration-layer)
7. [Architecture des bases de données](#7-architecture-des-bases-de-données)
8. [Event System interne](#8-event-system-interne)
9. [Règles de conception logicielle modernes](#9-règles-de-conception-logicielle-modernes)
10. [Sécurité & Cryptographie](#10-sécurité--cryptographie)
11. [Infrastructure & Déploiement](#11-infrastructure--déploiement)
12. [Ce qui est interdit](#12-ce-qui-est-interdit)
13. [Décisions Technologiques Évaluées](#13-décisions-technologiques-évaluées)

---

## 1. Philosophie architecturale

### 1.1 Décision fondamentale

**Monolithe Modulaire + Services Externes, préparé pour extraction progressive.**

Ce n'est pas un compromis. C'est le choix optimal pour ATANAX en 2026 :

- Le cœur ERP (Axions360 + Synchro) est un **monolithe modulaire** — une seule application, des modules isolés, un seul déploiement, une seule base de données transactionnelle.
- Les services lourds, financiers ou à haute concurrence (Dymmo, IA, Sync Engine, Notifications) sont des **services externes** dès le départ.
- L'architecture est **extractible** — chaque module est codé avec des frontières claires, comme s'il pouvait devenir un microservice demain.

### 1.2 Pourquoi pas 100% microservices dès le départ

| Problème microservices | Impact sur ATANAX |
|------------------------|-------------------|
| Transactions distribuées (Saga pattern) | Inacceptable pour un ERP OHADA — une vente doit mettre à jour stock + comptabilité atomiquement |
| Déploiement local complexe | Synchro doit tourner sur un serveur local en Guinée sans Kubernetes |
| Latence réseau inter-services | KDS et POS exigent < 50ms de réponse |
| Complexité DevOps massive | L'équipe doit avancer vite sur le produit, pas sur l'infrastructure |
| Traçabilité des bugs difficile | Un bug traversant 6 services est un cauchemar de diagnostic |

### 1.3 La phrase clé

> **On ne démarre pas en microservices, mais on code comme si chaque module pouvait en devenir un.**

Cela signifie :
- Frontières de domaine strictes entre modules
- Communication inter-modules uniquement via Events ou API interne
- Pas d'accès direct aux tables d'un autre module
- Chaque module a ses propres Services, Repositories et DTOs

### 1.5 Règle d'agnosticisme technologique

> **Toute règle architecturale qui doit survivre à un changement de framework ou de langage doit vivre dans la base de données, pas dans le code applicatif.**

L'écosystème ATANAX est intentionnellement multi-technologique — Laravel, Go, Spring Boot, Rust, C++, Python opèrent sur les mêmes données. Une règle codée dans un `BaseModel` Laravel est invisible pour un service Go. Une règle en base est universelle.

**Conséquences concrètes :**

| Règle | Mauvais endroit | Bon endroit |
|-------|----------------|-------------|
| Cette entité est auditée | Trait Laravel | `entity_registry.has_audit` |
| Cette entité utilise le verrouillage optimiste | BaseModel Eloquent | `entity_registry.has_row_version` |
| Ce statut s'appelle "En attente" en français | `resources/lang/fr.json` | `order_status_i18n` en base |
| Ce type de document est une facture | ENUM dans le schéma | `document_types` Lookup Table |

Le code applicatif **lit** les règles depuis la base. Il ne les **définit** pas.

Voir `AXIONS_SCHEMA_RULES.md` sections 1 et 3 pour l'implémentation complète avec exemples Go, Java, PHP.

### 1.6 Performance maximale — la vision ATANAX

L'ambition de performance maximale ne signifie pas "utiliser le langage le plus rapide partout". Elle signifie **utiliser le bon outil pour la bonne tâche** :

- **Rapidité de traitement métier** → Laravel (best-in-class pour ERP)
- **Concurrence et flux financiers** → Go
- **Sécurité mémoire et cryptographie** → Rust
- **Robustesse enterprise et reporting** → Spring Boot (JVM)
- **Performance native, hardware, offline** → C++
- **Intelligence artificielle** → Python
- **Frontend réactif et réutilisable** → React + TypeScript

---

## 2. Vue d'ensemble de l'écosystème

```
┌─────────────────────────────────────────────────────────────────────┐
│                         ATANAX ECOSYSTEM                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ www.*.com    │  │ app.*.com    │  │ Mobile (iOS / Android)   │  │
│  │ Sites vitr. │  │ Web App SPA  │  │ React Native / Capacitor │  │
│  │ Astro/Blade │  │ React + Vite │  │                          │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────────┘  │
│         │                 │                      │                  │
│         └─────────────────┴──────────────────────┘                  │
│                           │                                         │
│                    ┌──────▼───────┐                                 │
│                    │ API Gateway  │  ← authentification, rate limit │
│                    │ (Go / Nginx) │    routing, versioning          │
│                    └──────┬───────┘                                 │
│                           │                                         │
│         ┌─────────────────┼──────────────────────┐                  │
│         ▼                 ▼                      ▼                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ AXIONS CORE  │  │    DYMMO     │  │   SERVICES EXTERNES      │  │
│  │ (Monolithe   │  │  (Go / Java) │  │   IA · Notifs · Storage  │  │
│  │  Modulaire)  │  │              │  │   Sync · BI · Reports    │  │
│  │  Laravel     │  │              │  │                          │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────────┘  │
│         │                 │                      │                  │
│         └─────────────────┴──────────────────────┘                  │
│                           │                                         │
│                    ┌──────▼───────┐                                 │
│                    │ Data Layer   │                                 │
│                    │ PostgreSQL   │                                 │
│                    │ Redis        │                                 │
│                    │ ClickHouse   │                                 │
│                    │ S3/Object    │                                 │
│                    └──────────────┘                                 │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ OFFLINE LAYER (Local Network)                                │   │
│  │  Synchro Desktop (C++ / Tauri)  ←→  Local PostgreSQL        │   │
│  │  KDS · POS · Kiosque · Tablette serveur                     │   │
│  │  Rust Security Module (STC / Cryptographie locale)          │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Stack technologique par couche

### 3.1 Tableau de décision — Quel langage pour quoi

| Couche | Technologie | Justification |
|--------|-------------|---------------|
| **API Core (Axions ERP, Synchro Cloud)** | Laravel 11+ (PHP 8.3+) | Time-to-market imbattable, écosystème ERP, modules, permissions, queues |
| **Frontend Web App** | React 19 + TypeScript + Vite | SPA réutilisable Web/Desktop/Mobile, découplé du backend |
| **Site Vitrine** | Astro | SSG, SEO-first, ultra-rapide, zéro JS par défaut |
| **API Gateway** | Go (Fiber / Chi) | Performance native, faible latence, excellent pour proxy/routing |
| **Dymmo — Moteur financier** | Go | Goroutines pour haute concurrence, latence faible, sécurité des types |
| **Dymmo — Reporting bancaire** | Spring Boot (Java 21) | Robustesse JVM, écosystème bancaire (ISO 20022, SWIFT libs) |
| **Sécurité / Cryptographie / STC** | Rust | Mémoire sûre, zero-cost abstractions, post-quantique (ML-KEM, ML-DSA) |
| **Synchro Desktop (POS/KDS offline)** | C++ (Qt) ou Rust + Tauri | Performance native, accès hardware, fonctionnement sans internet |
| **Intelligence Artificielle (Quotient Engine)** | Python (FastAPI) | Écosystème ML/AI incomparable (PyTorch, scikit-learn, pandas) |
| **Sync Engine** | Go ou Rust | Synchronisation offline-online critique, performance maximale |
| **Mobile** | React Native ou Capacitor | Réutilisation du code React Web, une seule codebase |
| **Scripts DevOps / CLI** | Go | Binaires autonomes, cross-platform, rapides |
| **Analytics temps réel** | ClickHouse + Python | Colonnar storage, requêtes analytiques ultra-rapides |

### 3.2 Règle de sélection technologique

Avant d'introduire un nouveau langage ou framework dans la codebase, répondre à ces 3 questions :

1. **Existe-t-il un composant déjà dans la stack qui peut faire ce travail à 80% ?** Si oui, utiliser ce composant.
2. **Le gain de performance justifie-t-il la complexité opérationnelle supplémentaire ?** Quantifier le gain.
3. **L'équipe peut-elle maintenir ce code dans 3 ans ?** La performance d'un code que personne ne comprend est nulle.

### 3.3 Frontend — Architecture React SPA

```
app.axions360.com (React + TypeScript + Vite)
├── Découplé totalement du backend
├── Communique uniquement via REST API / JSON
├── Même codebase → Web App + Desktop (Tauri) + Mobile (Capacitor)
├── State management : Zustand (léger) ou Redux Toolkit (complexe)
├── Routing : React Router v7
├── UI Components : composants souverains ATANAX (pas shadcn comme seule source)
├── Forms : React Hook Form + Zod (validation TypeScript)
└── API Client : Axios avec interceptors pour auth + retry

Sites vitrine (Astro)
├── axions360.com
├── dymmo.io
├── synchro.app
└── atanax.com
```

**Pourquoi pas Laravel + Inertia ?**

Inertia crée un couplage serré entre le cycle de vie de Laravel et le rendu React. Cela empêche :
- La réutilisation du frontend sur Desktop et Mobile
- Le travail indépendant des équipes Frontend/Backend
- La stabilité face aux breaking changes Laravel/Inertia

---

## 4. Architecture du Monolithe Modulaire

### 4.1 Structure du projet Laravel

```
axions-core/                          ← Dépôt principal
├── app/
│   ├── Http/
│   │   ├── Controllers/              ← Controllers légers, délèguent aux Services
│   │   ├── Middleware/               ← Auth, Tenant, RateLimit
│   │   └── Resources/               ← API Resources (transformation JSON)
│   ├── Models/                       ← Eloquent Models (tables Core uniquement)
│   └── Providers/
│       └── EventServiceProvider.php  ← Mapping Events → Listeners
│
├── modules/                          ← Modules métier isolés
│   ├── Kernel/                       ← Axions Kernel (voir AXIONS_KERNEL.md)
│   ├── SynchroRestaurant/
│   │   ├── Models/
│   │   ├── Services/
│   │   ├── Events/
│   │   ├── Listeners/
│   │   ├── Http/
│   │   ├── Database/Migrations/
│   │   └── ModuleServiceProvider.php
│   ├── SynchroHotel/
│   ├── SynchroCommerce/
│   ├── AxionsHR/
│   ├── AxionsFinance/
│   ├── AxionsCRM/
│   ├── AxionsProjects/
│   └── AxionsImmobilier/
│
├── services/                         ← Services partagés inter-modules
│   ├── TenantService.php
│   ├── PermissionService.php
│   ├── AuditService.php
│   └── TransactionService.php        ← Interface vers table transactions (Core)
│
└── integrations/                     ← Integration Layer (voir section 6)
    ├── Delivery/
    ├── Payments/
    ├── Messaging/
    └── External/
```

### 4.2 Règles inter-modules

**Un module ne peut jamais :**

```php
// ❌ INTERDIT — SynchroRestaurant accède directement aux tables AxionsHR
use Modules\AxionsHR\Models\Contract;
$contract = Contract::where('employee_id', $id)->first();

// ❌ INTERDIT — Appel direct de méthode d'un autre module
app(AxionsHRService::class)->getEmployeeContracts($id);
```

**Un module doit toujours :**

```php
// ✅ CORRECT — Communication via Event
event(new EmployeeHiredEvent($employee));

// ✅ CORRECT — Lecture via Kernel (tables partagées)
$employee = Employee::find($id); // employees est une table Core

// ✅ CORRECT — Interface définie dans le Kernel
$this->transactionService->log([...]);
```

### 4.3 Structure d'un module

Chaque module respecte la même structure interne :

```
modules/SynchroRestaurant/
├── ModuleServiceProvider.php   ← Déclaration du module, dépendances, permissions
├── Models/
│   ├── Order.php               ← Eloquent, toujours scopé tenant_id
│   ├── MenuItem.php
│   └── Table.php
├── Services/
│   ├── OrderService.php        ← Logique métier (jamais dans le Controller)
│   └── KitchenService.php
├── Events/
│   ├── OrderPlacedEvent.php
│   └── OrderCompletedEvent.php
├── Listeners/
│   └── NotifyKitchenListener.php
├── DTOs/
│   ├── OrderDTO.php            ← Data Transfer Objects (pas d'arrays nus)
│   └── CreateOrderDTO.php
├── Http/
│   ├── Controllers/
│   └── Resources/              ← API Resources pour transformation JSON
├── Database/
│   └── Migrations/             ← Migrations propres au module
└── Tests/
    ├── Unit/
    └── Feature/
```

### 4.4 Service Layer obligatoire

Les Controllers ne contiennent **jamais** de logique métier :

```php
// ✅ CORRECT — Controller léger
class OrderController extends Controller
{
    public function store(CreateOrderRequest $request, OrderService $service): JsonResponse
    {
        $dto = CreateOrderDTO::fromRequest($request);
        $order = $service->create($dto);
        return new OrderResource($order);
    }
}

// ✅ CORRECT — Service avec logique métier
class OrderService
{
    public function create(CreateOrderDTO $dto): Order
    {
        return DB::transaction(function () use ($dto) {
            $order = Order::create([...]);
            $this->inventoryService->deduct($dto->items);
            $this->transactionService->log($order); // → table transactions (Core)
            event(new OrderPlacedEvent($order));
            return $order;
        });
    }
}
```

---

## 5. Services externes critiques

Ces services existent en dehors du monolithe Laravel dès le départ. Ils ont leur propre dépôt, leur propre déploiement, leur propre base de données si nécessaire.

### 5.1 Dymmo — Moteur financier

```
dymmo-engine/
├── Stack : Go (moteur principal) + Spring Boot (reporting bancaire)
├── Responsabilités :
│   ├── Orchestration des rails de paiement
│   ├── FX Engine (taux de change temps réel)
│   ├── Compliance (BCRG, BCEAO, FINTRAC)
│   ├── KYC/KYB pipeline
│   ├── Wallet management
│   └── Settlement Engine
├── Communication avec Axions Core :
│   ├── REST API sécurisée (mTLS)
│   └── Webhooks pour confirmations de paiement
└── Base de données : PostgreSQL dédié (isolation financière totale)
```

**Règle** : Axions Core ne traite jamais directement un paiement. Il délègue systématiquement à Dymmo via API.

### 5.2 Quotient Engine — IA & Scoring

```
quotient-engine/
├── Stack : Python (FastAPI + PyTorch + pandas)
├── Responsabilités :
│   ├── Scoring crédit (DymScore)
│   ├── Recommandations produit
│   ├── Prévisions de stock
│   ├── Détection d'anomalies financières
│   └── Analytics prédictifs
├── Communication : REST API + file d'attente async (Redis/Celery)
└── Base de données : PostgreSQL + ClickHouse (feature store)
```

### 5.3 Sync Engine — Synchronisation offline/online

```
sync-engine/
├── Stack : Go ou Rust
├── Responsabilités :
│   ├── Détection de conflits offline/online
│   ├── Merge de données
│   ├── File de synchronisation prioritaire
│   └── Gestion des partitions réseau (CAP theorem)
├── Protocole : Événements avec vector clocks
└── Déploiement : Edge (proche des clients) + Cloud central
```

### 5.4 Security Module — Cryptographie (STC)

```
security-module/
├── Stack : Rust
├── Responsabilités :
│   ├── Post-quantum cryptography (ML-KEM / ML-DSA)
│   ├── Sender Trace Capsule (STC) pour OfflinePay
│   ├── Signature Ed25519
│   ├── Validation TEE (Trusted Execution Environment)
│   └── Génération et rotation des clés
└── Déploiement : Local (embarqué dans Synchro Desktop) + Cloud
```

### 5.5 Notification Service

```
notification-service/
├── Stack : Go ou Laravel Queue Worker dédié
├── Canaux :
│   ├── SMS (Orange, MTN, Africell)
│   ├── WhatsApp Business API
│   ├── Email (SES / Postmark)
│   └── Push (Firebase FCM)
└── Garanties : At-least-once delivery, retry avec backoff exponentiel
```

---

## 6. Integration Layer

### 6.1 Principe

Les plateformes externes (Uber Eats, Orange Money, DoorDash, Mamdoo) ne sont **jamais** au centre. Ce sont des **canaux entrants ou sortants**. Le centre reste toujours :

```
Synchro → opération terrain
Axions ERP → structure groupe
Dymmo → flux financiers
```

### 6.2 Structure

```
integrations/
├── Delivery/
│   ├── UberEatsAdapter.php
│   ├── MamdooAdapter.php
│   └── DeliveryAdapterInterface.php   ← contrat commun
│
├── Payments/
│   ├── DymmoAdapter.php
│   ├── OrangeMoneyAdapter.php
│   ├── MTNMoMoAdapter.php
│   └── PaymentAdapterInterface.php
│
├── Messaging/
│   ├── WhatsAppAdapter.php
│   ├── SMSAdapter.php
│   └── MessagingAdapterInterface.php
│
└── Accounting/
    ├── SageExportAdapter.php
    └── AccountingAdapterInterface.php
```

### 6.3 Tables de traçabilité des intégrations

```sql
-- Registre des intégrations actives par tenant/store
external_integrations
├── id
├── tenant_id
├── store_id
├── provider          -- "mamdoo", "uber_eats", "orange_money"
├── type              -- "delivery", "payment", "messaging"
├── status            -- "active", "suspended", "error"
├── credentials       -- chiffré (AES-256-GCM), jamais en clair
├── webhook_url
├── webhook_secret    -- HMAC secret pour valider les webhooks entrants
├── last_sync_at
├── metadata          JSON NULL
└── timestamps

-- Traçabilité de chaque commande externe
external_orders
├── id
├── tenant_id
├── integration_id
├── internal_order_id -- → restaurant_orders.id
├── external_order_id -- ID chez Uber/Mamdoo
├── external_status
├── raw_payload       JSON   -- payload brut conservé pour audit/debug
├── synced_at
└── timestamps
```

### 6.4 Flux d'une commande externe (Mamdoo → KDS)

```
1. Mamdoo envoie un webhook POST /webhooks/mamdoo
2. Integration Layer valide la signature HMAC
3. Transforme le payload → CreateOrderDTO (format interne)
4. Appelle OrderService::create(dto, source: 'mamdoo')
5. OrderService crée l'order avec source='mamdoo', external_id=...
6. Émet OrderPlacedEvent
7. KDS Listener reçoit l'événement → affiche la commande
8. Finance Listener crée l'écriture → table transactions
```

Le KDS ne sait pas que la commande vient de Mamdoo. Il voit juste une commande.

---

## 7. Architecture des bases de données

### 7.1 Décision principale

**PostgreSQL comme base principale.** Pas MySQL. Pas MariaDB.

| Raison | Détail |
|--------|--------|
| ACID complet | Critique pour les données financières OHADA |
| JSONB natif | Indexable, requêtable, performant |
| Foreign Data Wrappers | Connexion à des sources externes sans ETL |
| Row Level Security | Multi-tenancy au niveau base de données |
| Partitionnement natif | Tables `audit_logs` et `transactions` peuvent être partitionnées par date |
| Extensions | `uuid-ossp`, `pgcrypto`, `pg_trgm` (recherche floue), `postgis` (géolocalisation) |
| Performances analytiques | Bien meilleur que MySQL pour les requêtes complexes |

### 7.2 Stack complète des données

```
┌─────────────────────────────────────────────┐
│              DATA LAYER                     │
├─────────────────┬───────────────────────────┤
│  PostgreSQL     │  OLTP principal           │
│  (primaire)     │  Toutes tables Core       │
│                 │  Modules ERP              │
│                 │  Dymmo (dédié séparé)     │
├─────────────────┼───────────────────────────┤
│  Redis          │  Cache applicatif         │
│                 │  Sessions                 │
│                 │  Rate limiting            │
│                 │  Queues Laravel           │
│                 │  Pub/Sub léger            │
├─────────────────┼───────────────────────────┤
│  ClickHouse     │  Analytics & BI           │
│                 │  Logs d'événements        │
│                 │  Métriques temps réel     │
│                 │  Rapports consolidés      │
├─────────────────┼───────────────────────────┤
│  S3 / MinIO     │  Fichiers (PDF, images)   │
│                 │  Backups                  │
│                 │  Exports                  │
├─────────────────┼───────────────────────────┤
│  SQLite         │  Local uniquement         │
│  (embarqué)     │  Synchro Desktop offline  │
│                 │  Sync vers PostgreSQL     │
└─────────────────┴───────────────────────────┘
```

### 7.3 Règles de conception base de données

#### Identifiants

```sql
-- Toute table a :
id      BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY  -- usage interne, JOINs
ulid    CHAR(26) UNIQUE NOT NULL                   -- usage externe, API, URLs

-- Jamais exposer id dans une URL ou réponse API publique
-- Toujours utiliser ulid en externe
```

#### Montants monétaires

```sql
-- TOUJOURS BIGINT en sous-unités
amount  BIGINT NOT NULL   -- 10050 = 100.50 USD, 15000 = 15 000 GNF

-- JAMAIS
amount  DECIMAL(10,2)     -- erreurs d'arrondi garanties
amount  FLOAT             -- approximation inacceptable en finance
amount  VARCHAR           -- absurde
```

#### Timestamps

```sql
-- Standard sur toutes les tables
created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
deleted_at  TIMESTAMP NULL      -- soft delete quand nécessaire (pas toujours)

-- Timestamps métier nommés explicitement
occurred_at    TIMESTAMP   -- quand l'événement a eu lieu (≠ created_at)
processed_at   TIMESTAMP   -- quand le système l'a traité
completed_at   TIMESTAMP   -- quand c'est terminé
```

#### Index

```sql
-- Règle : toute colonne utilisée dans WHERE, JOIN ou ORDER BY a un index
-- Index composites : la colonne la plus sélective en premier
CREATE INDEX idx_orders_tenant_status ON restaurant_orders(tenant_id, status);
CREATE INDEX idx_transactions_occurred ON transactions(tenant_id, occurred_at DESC);

-- Index partiels pour les requêtes fréquentes sur sous-ensembles
CREATE INDEX idx_orders_pending ON restaurant_orders(tenant_id, created_at)
    WHERE status = 'pending';
```

#### Partitionnement

Pour les tables volumineuses :

```sql
-- audit_logs et transactions partitionnées par mois
CREATE TABLE transactions (
    ...
) PARTITION BY RANGE (occurred_at);

CREATE TABLE transactions_2026_01 PARTITION OF transactions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

#### Chiffrement des données sensibles

```sql
-- Credentials, tokens, données personnelles sensibles
credentials  TEXT NOT NULL  -- chiffré AES-256-GCM avant insertion, jamais en clair en DB
```

Le chiffrement se fait **au niveau applicatif** (Laravel), pas au niveau base de données uniquement.

### 7.4 Migrations — Règles strictes

```
Numérotation : YYYY_MM_DD_HHMMSS_description_action.php
Exemple : 2026_06_15_143022_create_restaurant_orders_table.php

Ordre d'exécution obligatoire :
1. axions-kernel (tables core)
2. axions-auth (roles, permissions)
3. modules Synchro
4. modules Axions ERP
5. Dymmo
6. Integration Layer

Règles :
- Idempotentes (Schema::createIfNotExists)
- Jamais de DROP en production sans migration de données
- Jamais de modification de colonne sans backward compatibility d'abord
- Toute migration irréversible requiert une approbation explicite
```

### 7.5 Read vs Write — CQRS léger

Pour les modules à fort volume (transactions, analytics) :

```
Écriture → PostgreSQL primaire
Lecture analytique → PostgreSQL replica ou ClickHouse

Jamais de requête analytique lourde sur le primaire en production.
```

---

## 8. Event System interne

### 8.1 Principe

Le monolithe utilise un **Event Bus interne** (Laravel Events) pour la communication entre modules. Pas de Kafka, pas de RabbitMQ pour la communication intra-application.

```
Module A émet un Event
    ↓
Laravel EventServiceProvider route l'Event
    ↓
Module B reçoit via Listener (async si nécessaire)
    ↓
Module B réagit sans jamais appeler Module A directement
```

### 8.2 Convention de nommage des Events

```php
// Format : {Module}\Events\{Entite}{Action}Event
Modules\SynchroRestaurant\Events\OrderPlacedEvent
Modules\SynchroRestaurant\Events\OrderCompletedEvent
Modules\AxionsHR\Events\EmployeeHiredEvent
Modules\AxionsFinance\Events\TransactionRecordedEvent
Modules\Dymmo\Events\PaymentConfirmedEvent
```

### 8.3 Structure d'un Event

Chaque Event Laravel est associé à un **opscode** — la clé universelle du système.

```php
class OrderPlacedEvent
{
    // Opscode associé — doit exister dans le catalogue opscodes
    public const OPSCODE = 'synchro.restaurant.order.created';

    public function __construct(
        public readonly string             $tenantId,
        public readonly string             $storeId,
        public readonly string             $orderId,
        public readonly string             $orderUlid,
        public readonly int                $totalAmount,       // BIGINT sous-unités
        public readonly string             $currencyCode,
        public readonly string             $source,            // "pos", "mamdoo", "online"
        public readonly \DateTimeImmutable $occurredAt,
    ) {}
}
```

**Règles sur les Events :**
- Immutables (readonly properties)
- Contiennent des IDs et valeurs primitives, jamais des objets Eloquent
- Ont un `occurredAt` explicite (pas seulement `created_at`)
- Déclarent une constante `OPSCODE` référençant le catalogue `opscodes`
- Sont persistés dans `domain_events` pour rejouabilité

### 8.4 Table de persistance des événements domaine

Voir `AXIONS_KERNEL.md` section 2.12 pour le schéma complet.

```sql
domain_events
├── id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
├── ulid            CHAR(26) UNIQUE NOT NULL
├── tenant_id       BIGINT UNSIGNED NOT NULL
├── opscode_code    VARCHAR(128) NOT NULL REFERENCES opscodes(code)
├── aggregate_type  VARCHAR(100) NOT NULL    -- "Order", "Employee"
├── aggregate_id    BIGINT UNSIGNED NOT NULL
├── payload         JSONB NOT NULL
├── occurred_at     TIMESTAMP WITH TIME ZONE NOT NULL
├── processed_at    TIMESTAMP WITH TIME ZONE NULL
```

### 8.5 Opscodes — Langage universel du système

Tout événement, audit, notification et transaction est relié via un **opscode**. Voir `AXIONS_SCHEMA_RULES.md` section 5 pour la définition complète.

```
opscode                  = ce qui s'est passé (événement métier)
operation_code           = l'impact financier sur le grand livre

synchro.restaurant.order.paid  →  audit_log.opscode_code
                               →  domain_events.opscode_code
                               →  transactions.source_opscode_code
                               →  transactions.operation_code = MERCHANT_PAYMENT_CREDIT
```

---

## 9. Règles de conception logicielle modernes

### 9.1 Principes SOLID appliqués

**Single Responsibility** : Un Service fait une chose. `OrderService` crée des commandes. Il ne gère pas les notifications (c'est `NotificationService`) ni la comptabilité (c'est un Listener de `OrderPlacedEvent`).

**Open/Closed** : Les adaptateurs d'intégration implémentent des interfaces. Ajouter Mamdoo ne modifie pas le code existant, on ajoute `MamdooAdapter implements DeliveryAdapterInterface`.

**Liskov Substitution** : Tout `PaymentAdapterInterface` est interchangeable. Le OrderService ne sait pas si le paiement passe par Dymmo, Orange Money ou MTN.

**Interface Segregation** : Les interfaces sont petites et spécifiques. Pas une `MegaServiceInterface` avec 30 méthodes.

**Dependency Injection** : Jamais de `new Service()` dans le code applicatif. Tout passe par le conteneur IoC de Laravel.

### 9.2 Data Transfer Objects (DTOs)

**Jamais passer des arrays bruts entre couches.** Les DTOs sont typés, validés, immutables.

```php
// ❌ INTERDIT
$orderService->create($request->all());

// ✅ CORRECT
final class CreateOrderDTO
{
    public function __construct(
        public readonly string $tenantId,
        public readonly string $storeId,
        public readonly string $contactId,
        /** @var array<OrderItemDTO> */
        public readonly array  $items,
        public readonly string $source,
    ) {}

    public static function fromRequest(CreateOrderRequest $request): self
    {
        return new self(
            tenantId:  auth()->user()->tenant_id,
            storeId:   $request->validated('store_id'),
            contactId: $request->validated('contact_id'),
            items:     OrderItemDTO::collection($request->validated('items')),
            source:    $request->validated('source', 'pos'),
        );
    }
}
```

### 9.3 Repository Pattern (pour les requêtes complexes)

```php
// Les requêtes complexes → Repository (pas dans le Service, pas dans le Controller)
interface OrderRepositoryInterface
{
    public function findPendingByStore(string $storeId): Collection;
    public function findByExternalId(string $provider, string $externalId): ?Order;
    public function getDailyRevenue(string $storeId, Carbon $date): int;
}
```

### 9.4 Validation stricte

```php
// Validation en entrée — FormRequest Laravel
class CreateOrderRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'store_id'       => ['required', 'ulid', 'exists:stores,ulid'],
            'items'          => ['required', 'array', 'min:1'],
            'items.*.id'     => ['required', 'ulid', 'exists:restaurant_menu_items,ulid'],
            'items.*.qty'    => ['required', 'integer', 'min:1', 'max:99'],
            'source'         => ['required', Rule::in(['pos', 'online', 'mamdoo', 'uber_eats'])],
        ];
    }
}
```

### 9.5 Error Handling

```php
// Exceptions métier typées — jamais de Exception générique
class OrderNotFoundException extends DomainException {}
class InsufficientStockException extends DomainException
{
    public function __construct(
        public readonly string $itemId,
        public readonly int $requested,
        public readonly int $available,
    ) {
        parent::__construct("Stock insuffisant pour l'article {$itemId}");
    }
}

// Handler global → réponse API cohérente
// Toutes les réponses d'erreur suivent le même format JSON
{
    "error": {
        "code":    "INSUFFICIENT_STOCK",
        "message": "Stock insuffisant pour l'article xyz",
        "details": { "requested": 5, "available": 2 }
    }
}
```

### 9.6 API Design — REST + JSON

```
Conventions :
- Noms de ressources au pluriel : /orders, /employees, /stores
- Identifiants : ULID dans les URLs, jamais d'ID entier
- Versioning : /api/v1/orders (dans l'URL, pas dans le header)
- Méthodes HTTP sémantiques : GET, POST, PUT, PATCH, DELETE
- Codes HTTP corrects : 200, 201, 204, 400, 401, 403, 404, 422, 429, 500

Format de réponse standard :
{
    "data": { ... },          // pour les succès
    "meta": {                 // pagination si applicable
        "current_page": 1,
        "per_page": 25,
        "total": 142
    }
}

Format d'erreur standard :
{
    "error": {
        "code":    "VALIDATION_ERROR",
        "message": "Les données fournies sont invalides.",
        "details": { "field": ["message d'erreur"] }
    }
}
```

### 9.7 Testing obligatoire

```
Coverage minimum par type :
- Services métier : 90%+ (tests unitaires avec mocks)
- API Endpoints : 80%+ (tests feature)
- Events/Listeners : 85%+ (tests d'intégration)
- Integration Adapters : 75%+ (tests avec mocks externes)

Types de tests :
Unit/          → Services, DTOs, Value Objects (rapides, isolés)
Feature/       → Endpoints API complets (avec DB de test)
Integration/   → Flux complets multi-modules
Architecture/  → Vérification que les modules ne se couplent pas
```

---

## 10. Sécurité & Cryptographie

### 10.1 Authentification

```
Web App / Mobile :
- JWT (access token 15min) + Refresh Token (30 jours, rotation)
- Stockage : HttpOnly cookie (pas localStorage)

API externe (partenaires) :
- API Keys avec scopes (lecture/écriture/admin)
- Rotation obligatoire tous les 90 jours

Service-to-Service (interne) :
- mTLS (mutual TLS) entre services critiques
- Dymmo ↔ Axions Core : mTLS obligatoire
```

### 10.2 Cryptographie post-quantique (module Rust)

```
Algorithmes retenus (NIST Post-Quantum standards) :
- ML-KEM (Kyber) : échange de clés
- ML-DSA (Dilithium) : signatures numériques
- SPHINCS+ : signatures stateless (backup)

Utilisés par :
- STC (Sender Trace Capsule) — OfflinePay
- Signatures de documents contractuels
- Chiffrement des credentials en base
```

### 10.3 Règles de sécurité des données

```
Données en transit : TLS 1.3 minimum, HSTS obligatoire
Données au repos : AES-256-GCM pour les données sensibles
Credentials tiers : jamais en clair en DB, toujours chiffrés au niveau app
Logs : jamais de données personnelles dans les logs (masquage automatique)
Backups : chiffrés, testés mensuellement (restore drill)
```

---

## 11. Infrastructure & Déploiement

### 11.1 Environnements

```
local       → Docker Compose (PostgreSQL + Redis + services)
staging     → Réplique exacte de production, données anonymisées
production  → AWS (primaire) + OVHcloud (Europe/Afrique)
```

### 11.2 Déploiement Cloud vs Local

```
Cloud (Axions Core, Dymmo, Services) :
- Containerisé (Docker)
- Orchestration : Kubernetes (quand la scale le justifie) ou ECS
- CI/CD : GitHub Actions
- Zero-downtime deployment : rolling updates

Local (Synchro Desktop) :
- Installateur autonome (Windows / Linux)
- PostgreSQL embarqué local
- Sync Engine pour synchronisation périodique
- Mise à jour automatique silencieuse
```

### 11.3 Observabilité

```
Logs structurés (JSON) → centralisés (Loki / CloudWatch)
Métriques → Prometheus + Grafana
Tracing distribué → OpenTelemetry (corrèle requêtes cross-services)
Alertes → PagerDuty / OpsGenie pour incidents critiques
Uptime → Monitoring externe (Better Uptime)
```

---

## 12. Ce qui est interdit

| ❌ Interdit | Raison | ✅ Correct |
|-------------|--------|-----------|
| Logique métier dans un Controller | Intenable à maintenir | Service Layer |
| Array brut entre couches | Pas de typage, bugs silencieux | DTO typé |
| `new Service()` dans le code app | Couplage fort | Injection de dépendances |
| Accès direct aux tables d'un autre module | Viole les frontières de domaine | Events ou API interne |
| `SELECT *` | Performance, surface d'exposition | Sélectionner uniquement les colonnes nécessaires |
| DECIMAL/FLOAT pour les montants | Erreurs d'arrondi en finance | BIGINT en sous-unités |
| ID entier dans les URLs | Sécurité (enumeration attack) | ULID |
| Credentials en clair en base | Faille de sécurité critique | Chiffrement AES-256-GCM |
| Données personnelles dans les logs | RGPD / vie privée | Masquage automatique |
| Tests absents sur les Services | Régression garantie | Coverage 90%+ |
| Laravel + Inertia pour Axions Core | Couplage frontend/backend | React SPA séparé + Laravel API |
| Kafka pour communication intra-monolithe | Complexité inutile | Laravel Events internes |
| Microservices pour modules ERP | Transactions distribuées, déploiement local impossible | Monolithe modulaire |
| Float pour coordonnées GPS | Imprécision | DECIMAL(10,8) et DECIMAL(11,8) |
| Secrets dans le code source | Faille critique | Variables d'environnement + Vault |
| ENUM hardcodé dans le schéma | Inextensible, pas de i18n possible | Lookup Table `{entity}_statuses` + i18n sidecar |
| Texte affiché en dur dans le code | Incompatible avec le multilingue | Opscode translations ou i18n |
| `user_id NOT NULL` sur `employees` | Un employé peut exister sans accès système | `user_id NULL` — journaliers, employés sans compte |
| Événement sans opscode déclaré | Pas de traçabilité universelle ni de traduction | Opscode dans le catalogue kernel + translations |
| Données PII en clair en base | Faille de sécurité critique | AES-256-GCM + blind index + key_version_id |

---

## Annexe A — Résumé des décisions architecturales

| Décision | Choix retenu | Raison principale |
|----------|-------------|-------------------|
| Architecture ERP | Monolithe modulaire | Cohérence transactionnelle, déploiement local |
| Architecture Dymmo | Service externe (Go) | Isolation financière, haute concurrence |
| Frontend | React SPA séparé | Réutilisabilité Web/Desktop/Mobile |
| Site vitrine | Astro | SEO, performance, zéro runtime JS |
| Base de données principale | PostgreSQL | ACID, JSONB, Row Level Security, partitionnement |
| Cache | Redis | Performance, queues, sessions |
| Analytics | ClickHouse | Colonnar, requêtes analytiques rapides |
| Cryptographie sensible | Rust | Mémoire sûre, post-quantique |
| Desktop offline | C++ (Qt) ou Tauri | Performance native, accès hardware |
| IA | Python (FastAPI) | Écosystème ML incomparable |
| Event Bus | Laravel Events (interne) | Simplicité, pas de broker externe |
| Identifiants publics | ULID | Ordonnés dans le temps, URL-safe |
| Montants monétaires | BIGINT sous-unités | Précision absolue, zéro arrondi |
| Communication inter-services | REST + mTLS | Standard, sécurisé, documentable |
| Statuts/types/catégories | Lookup Tables + i18n | Extensible, multilingue, zéro ENUM hardcodé |
| Événements système | Opscodes + translations | Langage universel pour audit/notif/i18n |
| Impact financier | Operation Codes | Catalogue Dymmo/Finance — DEBIT/CREDIT/NEUTRAL |
| Identité humaine | people → users / employees / contacts | Un employé peut exister sans accès système |
| Multilingue | Activé par tenant + store | Chaque tenant/store choisit ses langues |
| Règles d'architecture des entités | `entity_registry` en base | Agnostique au framework — survit à tout changement de service |
| Contrat d'entité | Entity Contract V1 (Juin 2026) | Standard versionné — toute table entité s'y conforme |
| Verrouillage optimiste | `row_version INTEGER DEFAULT 0` | Sans ambiguïté, conditionnel selon `entity_registry.has_row_version` |

---

## Annexe B — Checklist architecture pour chaque nouvelle fonctionnalité

Avant de coder une nouvelle fonctionnalité :

- [ ] Dans quel module appartient cette fonctionnalité ?
- [ ] Utilise-t-elle des tables Core ou des tables propres au module ?
- [ ] Entity Contract V1 respecté (id, ulid, tenant_id, status_code via Lookup, row_version si Tier 1/2, timestamps WITH TIME ZONE) ?
- [ ] Entrée créée dans `entity_registry` avec le bon tier et les bonnes protections ?
- [ ] Les montants sont-ils en BIGINT sous-unités ?
- [ ] Les entités publiques ont-elles un ULID ?
- [ ] Aucun ENUM hardcodé — Lookup Table + i18n sidecar ?
- [ ] Le contenu visible utilisateur a-t-il une table `_translations` ou opscode translation ?
- [ ] Les nouveaux événements ont-ils un opscode déclaré avec traductions FR/EN minimum ?
- [ ] Les opérations financières ont-elles un `operation_code` dans le catalogue ?
- [ ] La logique est-elle dans un Service (pas dans le Controller) ?
- [ ] Les données entre couches passent-elles par un DTO ?
- [ ] La communication avec un autre module passe-t-elle par un Event avec opscode ?
- [ ] Les transactions financières créent-elles une entrée dans `transactions` avec `operation_code` + `source_opscode_code` ?
- [ ] Les actions sensibles sont-elles auditées dans `audit_logs` avec `opscode_code` ?
- [ ] Les opérations monétaires ont-elles une `idempotency_key` avec contrainte UNIQUE ?
- [ ] Les tests unitaires couvrent-ils le Service à 90%+ ?
- [ ] L'API respecte-t-elle le format de réponse standard ?
- [ ] Aucun secret n'est codé en dur ?

---

## 13. Décisions Technologiques Évaluées

> Cette section documente toutes les technologies évaluées pour l'écosystème ATANAX, avec leur statut, leur justification et leur condition de révision.
>
> **Objectif** : éviter de rouvrir un débat déjà tranché. Quand une condition de révision est remplie, la décision est réexaminée avec les nouvelles données. Pas avant.

---

### 13.1 Stockage & Bases de données

---

#### PostgreSQL — ADOPTÉ (Core)

**Statut** : Adopté. Base de données principale de tout l'écosystème.

**Contexte d'utilisation** :
- Toutes les tables Core du Kernel (tenants, users, employees, transactions...)
- Tous les modules Synchro et Axions ERP
- Base dédiée séparée pour Dymmo (isolation financière)
- SQLite embarqué localement pour Synchro Desktop offline (variante légère, même paradigme)

**Justification** :
PostgreSQL est la seule base de données open source qui réunit simultanément : ACID complet (critique pour un ERP financier OHADA), JSONB indexable (colonne `metadata`), Row Level Security (renforcement de l'isolation multi-tenant au niveau base), partitionnement natif par date (audit_logs, domain_events), extensions pgcrypto et pg_trgm (recherche floue sur noms africains), et des performances analytiques supérieures à MySQL/MariaDB sur les requêtes complexes de reporting.

**Condition de révision** : Aucune. PostgreSQL est un choix structurant à long terme, pas une décision à réévaluer.

---

#### Redis — ADOPTÉ (Cache & Queues)

**Statut** : Adopté.

**Contexte d'utilisation** :
- Cache applicatif (sessions, résultats de requêtes fréquentes)
- Queue Laravel (jobs asynchrones : notifications, sync, exports)
- Rate limiting au niveau API Gateway
- Pub/Sub léger intra-service

**Justification** :
Redis est le standard pour ces cas d'usage. In-memory, sub-milliseconde, protocole simple. Aucune alternative ne justifie la complexité supplémentaire à ce stade.

**Condition de révision** : Si le volume de queues dépasse ce que Redis peut absorber en mémoire, évaluer Redis Cluster ou migration vers un broker dédié.

---

#### ClickHouse — ADOPTÉ (Analytics)

**Statut** : Adopté pour la couche analytics uniquement.

**Contexte d'utilisation** :
- Rapports consolidés multi-stores (revenus, stocks, tendances)
- Métriques Quotient Engine (feature store pour l'IA)
- Dashboard analytique temps réel pour les groupes multi-entités

**Justification** :
Le stockage colonnaire de ClickHouse est 10 à 100x plus rapide que PostgreSQL pour les requêtes analytiques agrégées sur de grandes périodes. Séparer OLTP (PostgreSQL) et OLAP (ClickHouse) protège les performances transactionnelles — une requête analytique lourde ne peut pas bloquer les insertions de ventes en cours.

**Règle d'or** : Aucune requête analytique lourde (agrégations > 100k lignes, rapports multi-mois) ne doit s'exécuter sur le PostgreSQL primaire en production.

**Condition de révision** : Aucune. L'architecture OLTP/OLAP séparée est une décision structurante.

---

#### ScyllaDB — DIFFÉRÉ

**Statut** : Évalué, non adopté. Réévaluation conditionnelle.

**Ce que c'est** : Base de données NoSQL compatible Cassandra, écrite en C++, conçue pour les volumes extrêmes avec latence constante sous charge massive.

**Pourquoi évalué** : Candidat pour le stockage des `audit_logs` et `domain_events` à très grand volume — tables append-only sans relations, parfaitement adaptées au modèle wide-column de ScyllaDB.

**Pourquoi différé** :
PostgreSQL partitionné par date couvre confortablement `audit_logs` jusqu'à plusieurs centaines de millions de lignes. Introduire ScyllaDB aujourd'hui ajoute une complexité opérationnelle (nouveau paradigme de requêtes, pas de SQL standard, pas de JOINs) sans bénéfice mesurable à l'échelle actuelle.

**Condition de révision** : Si `audit_logs` ou `domain_events` dépassent **50 millions de lignes par mois** sur le primaire PostgreSQL ET que les performances de lecture commencent à dégrader, ScyllaDB devient candidat pour ces deux tables uniquement — pas pour le reste du système.

---

#### PouchDB / CouchDB — REJETÉ

**Statut** : Évalué, rejeté définitivement pour le cœur d'Axions360.

**Ce que c'est** : PouchDB est une base JavaScript offline-first qui synchronise avec CouchDB via un protocole de réplication natif gérant les conflits automatiquement.

**Pourquoi évalué** : Le modèle de synchronisation offline/online de CouchDB est élégant et éprouvé — il aurait pu remplacer notre Sync Engine custom.

**Pourquoi rejeté** :

1. **Incompatibilité avec l'Entity Contract** : CouchDB est document-oriented. Il n'y a pas de schéma, pas de relations, pas de contraintes de clé étrangère. Notre modèle de données repose sur des relations strictes (people → users → employees, tenant_id partout).

2. **Perte de contrôle sur la résolution de conflits** : CouchDB résout les conflits de réplication par un algorithme déterministe générique. Pour Axions360, un conflit sur une commande de restaurant a des règles métier spécifiques (la version avec le paiement confirmé gagne toujours). Ce niveau de contrôle nécessite un Sync Engine custom.

3. **Pas de SQL** : Toute la couche reporting, les Lookup Tables, les opscodes, les jointures entre modules — tout cela repose sur SQL. CouchDB ne parle pas SQL.

**Condition de révision** : Aucune. Le rejet est définitif pour le cœur ERP.

---

### 13.2 Communication & Protocoles

---

#### REST / HTTPS — ADOPTÉ (Standard)

**Statut** : Adopté. Protocole par défaut pour toute communication API.

**Contexte d'utilisation** :
- API publique (frontend React, partenaires externes, intégrations)
- Communication inter-services standards (Axions Core ↔ services externes)
- Webhooks entrants (plateformes de livraison, passerelles de paiement)

**Justification** :
REST/JSON est le protocole universel. Il est lisible, debuggable, documentable (OpenAPI), compatible avec tous les clients existants et futurs. Chaque service de l'écosystème — Laravel, Go, Spring Boot, Rust, C++ — parle REST nativement. La simplicité opérationnelle prime sur l'optimisation prématurée.

**Condition de révision** : Aucune pour l'API publique. Voir gRPC pour les cas inter-services critiques.

---

#### gRPC — ENVISAGÉ (Inter-services critiques)

**Statut** : Évalué positivement. Adoption conditionnelle sur les chemins critiques uniquement.

**Ce que c'est** : Protocole RPC haute performance de Google. Communication binaire via Protocol Buffers (Protobuf), contrats typés générés automatiquement, 5 à 10x plus rapide que REST/JSON, streaming natif bidirectionnel.

**Contexte d'utilisation envisagé** :
- `Dymmo ↔ Axions Core` : appels de validation de paiement latence-sensibles (< 10ms)
- `Sync Engine ↔ Synchro Desktop/Mobile` : synchronisation de données volumineuses
- Communication interne haute fréquence entre services Go

**Pourquoi pas maintenant** :
gRPC introduit une complexité opérationnelle significative : génération de code Protobuf à maintenir, debugging moins immédiat que REST, outillage plus lourd. À ce stade, la latence REST est acceptable pour tous les cas d'usage actuels.

**Condition d'adoption** : Quand un chemin inter-services dépasse **500 requêtes/seconde** de façon soutenue ET que la latence REST devient un goulot d'étranglement mesurable, gRPC est adopté sur ce chemin spécifique. Pas sur l'ensemble du système.

---

#### GraphQL — OPTIONNEL

**Statut** : Évalué. Non prioritaire. Adoption possible uniquement côté frontend.

**Ce que c'est** : Langage de requête API qui permet au client de demander exactement les champs dont il a besoin, en une seule requête.

**Contexte d'utilisation envisagé** :
- Dashboard Axions Web App — écrans qui agrègent beaucoup d'entités différentes en une requête

**Pourquoi non prioritaire** :
GraphQL résout un problème réel (over-fetching, under-fetching) mais introduit une complexité backend non négligeable : resolver N+1 queries, gestion du cache, surface d'attaque plus large, outillage de monitoring différent. REST bien conçu avec des endpoints spécialisés résout le même problème sans cette complexité.

**Condition d'adoption** : Si le frontend React identifie plus de 5 écrans avec des besoins d'agrégation complexes que REST ne peut pas servir efficacement, GraphQL est évalué pour la couche BFF (Backend For Frontend) uniquement — pas pour l'API publique ni pour Dymmo.

---

### 13.3 Event Streaming & Bus de messages

---

#### Laravel Events (internes) — ADOPTÉ (Intra-monolithe)

**Statut** : Adopté. Bus d'événements par défaut pour la communication intra-monolithe.

**Contexte d'utilisation** :
- Communication entre modules Axions Core (Synchro → Finance, HR → Audit...)
- Déclenchement des Listeners asynchrones (notifications, indexation, sync partielle)

**Justification** :
Pour un monolithe modulaire, Laravel Events est le choix optimal. Pas de broker externe à maintenir, pas de latence réseau, transactions atomiques possibles (un event peut être émis dans une transaction DB et annulé si elle échoue). La simplicité opérationnelle est un avantage critique dans la phase actuelle.

**Condition de révision** : Voir Redpanda pour la communication inter-services externes.

---

#### Redpanda — DIFFÉRÉ (Inter-services à volume)

**Statut** : Évalué positivement. Adoption conditionnelle pour la communication inter-services à grand volume.

**Ce que c'est** : Broker de streaming d'événements compatible API Kafka, écrit en C++, déployable sur un seul nœud sans ZooKeeper ni JVM. Redpanda est préféré à Kafka pour ATANAX pour trois raisons : déploiement plus simple (critique pour les contraintes infrastructure africaine), latence plus faible, et consommation mémoire réduite.

**Pourquoi Redpanda plutôt que Kafka** :
Kafka nécessite ZooKeeper (ou KRaft en mode récent), une JVM, et un cluster d'au moins 3 brokers pour la haute disponibilité. Redpanda tourne sur un seul nœud avec les mêmes garanties. Pour une infrastructure qui doit fonctionner en Afrique de l'Ouest avec des contraintes hardware, Redpanda est plus pragmatique.

**Contexte d'utilisation envisagé** :
- `Axions Core → Quotient Engine` : flux de données pour l'entraînement et l'inférence ML
- `Axions Core → Sync Engine` : événements de synchronisation à distribuer
- `Dymmo → Axions Core` : confirmations de paiement à traiter en streaming
- Event sourcing si `domain_events` PostgreSQL atteint ses limites

**Condition d'adoption** : Quand le volume d'événements inter-services dépasse **100 000 events/jour** de façon soutenue, OU quand plus de 3 services externes doivent consommer le même flux d'événements.

---

#### Apache Kafka — NON RETENU

**Statut** : Évalué. Remplacé par Redpanda si un broker externe devient nécessaire.

**Justification** : Kafka offre les mêmes garanties que Redpanda avec une complexité opérationnelle supérieure (ZooKeeper ou KRaft, JVM, cluster obligatoire). Pour ATANAX, Redpanda couvre tous les cas d'usage de Kafka avec moins de friction. Si Redpanda atteint ses limites, la migration vers Kafka est triviale — l'API est compatible.

**Condition de révision** : Si Redpanda montre des limitations spécifiques à l'échelle ATANAX, Kafka est réévalué. Pas avant.

---

### 13.4 API Gateway & Service Mesh

---

#### KrakenD — ADOPTÉ (API Gateway)

**Statut** : Adopté pour la couche API Gateway.

**Ce que c'est** : API Gateway stateless ultra-performant, écrit en Go, configuré par fichiers déclaratifs (JSON/YAML). Aucune base de données requise pour fonctionner.

**Contexte d'utilisation** :
- Point d'entrée unique pour toutes les APIs publiques d'Axions360
- Authentification JWT, rate limiting, routing vers les services
- Agrégation de réponses multi-services en une seule réponse client
- Versioning des APIs (`/api/v1/`, `/api/v2/`)

**Justification** :
KrakenD est écrit en Go — cohérent avec notre API Gateway Go prévu. Stateless par design, ce qui simplifie le déploiement et la scalabilité horizontale. La configuration déclarative permet de versionner la configuration de la gateway dans Git comme le reste du code. Pas de base de données à maintenir contrairement à Kong.

**Condition de révision** : Si KrakenD atteint des limitations fonctionnelles spécifiques non couvertes par sa configuration, Kong ou un gateway custom Go sont évalués.

---

#### Envoy — DIFFÉRÉ (Service Mesh)

**Statut** : Évalué. Non adopté à ce stade.

**Ce que c'est** : Proxy L7 haute performance, utilisé comme sidecar dans les architectures service mesh (Istio, etc.). Gère le mTLS inter-services, le circuit breaking, l'observabilité au niveau réseau.

**Pourquoi différé** :
Envoy est pertinent dans une architecture Kubernetes avec de nombreux microservices. Dans le modèle actuel d'ATANAX (monolithe modulaire + services externes limités), Envoy ajoute une couche de complexité sans bénéfice proportionnel. Le mTLS inter-services est géré directement au niveau des certificats entre services.

**Condition d'adoption** : Si l'écosystème dépasse **8 services indépendants** en production ET que la gestion du mTLS et de l'observabilité inter-services devient un problème opérationnel réel.

---

### 13.5 Offline & Synchronisation

---

#### Sync Engine Custom (Go/Rust) — ADOPTÉ

**Statut** : Adopté. Composant critique développé en interne.

**Justification** :
La synchronisation offline/online d'Axions360 a des règles métier spécifiques que les solutions génériques ne peuvent pas exprimer. Exemple : un conflit sur une commande de restaurant doit être résolu en faveur de la version avec paiement confirmé, pas de la version la plus récente. Ce niveau de contrôle nécessite un Sync Engine custom.

Le Sync Engine est déployé en deux modes :
- **Cloud** : reçoit les deltas des clients, résout les conflits, distribue les mises à jour
- **Embarqué** : module léger dans Synchro Desktop qui gère la file de synchronisation locale

---

#### WatermelonDB — ADOPTÉ (Mobile uniquement)

**Statut** : Adopté pour Synchro Mobile (React Native / Capacitor).

**Ce que c'est** : Base de données offline-first pour React Native. SQLite sous le capot, API réactive, synchronisation via protocole custom.

**Contexte d'utilisation** :
- Tablettes serveurs en salle (prises de commande offline)
- Application manager mobile (consultation hors connexion)
- Kiosque client React Native

**Pourquoi pas pour Desktop** :
WatermelonDB est une bibliothèque JavaScript. Synchro Desktop est en C++ ou Tauri. Pour le Desktop, SQLite/PostgreSQL embarqué est utilisé directement.

**Condition de révision** : Si WatermelonDB montre des limitations sur les appareils Android d'entrée de gamme (courants en Afrique de l'Ouest), SQLite direct avec synchronisation custom est envisagé.

---

### 13.6 Tableau de décision — Vue consolidée

| Technologie | Statut | Contexte | Condition de révision |
|-------------|--------|----------|-----------------------|
| PostgreSQL | ✅ Adopté | Core ERP, Dymmo, modules | Aucune |
| SQLite (embarqué) | ✅ Adopté | Synchro Desktop offline | Aucune |
| Redis | ✅ Adopté | Cache, queues, sessions | Volume mémoire dépassé |
| ClickHouse | ✅ Adopté | Analytics, BI, feature store IA | Aucune |
| REST / HTTPS | ✅ Adopté | API publique, inter-services standard | Aucune |
| Laravel Events | ✅ Adopté | Bus intra-monolithe | Aucune |
| KrakenD | ✅ Adopté | API Gateway public | Limitations fonctionnelles |
| Sync Engine custom | ✅ Adopté | Offline/online Synchro | Aucune |
| WatermelonDB | ✅ Adopté | Synchro Mobile uniquement | Perf Android entrée de gamme |
| gRPC | ⏳ Envisagé | Inter-services critiques latence | > 500 req/s soutenu |
| Redpanda | ⏳ Différé | Streaming inter-services volume | > 100k events/jour |
| GraphQL | ⚠️ Optionnel | BFF frontend si nécessaire | > 5 écrans agrégation complexe |
| Envoy | ⏳ Différé | Service mesh | > 8 services en prod |
| ScyllaDB | ⏳ Différé | audit_logs volume extrême | > 50M lignes/mois |
| Apache Kafka | ❌ Non retenu | Remplacé par Redpanda | Si Redpanda insuffisant |
| PouchDB / CouchDB | ❌ Rejeté | Incompatible Entity Contract | Aucune révision |

---

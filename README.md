# Açucena Fleet & Logistics Manager

> Mobile application for fleet operations management with a security-first architecture.  
> Built from scratch to replace a paper-and-spreadsheet workflow for a real water distribution business in rural Pernambuco, Brazil.

[![Platform](https://img.shields.io/badge/Platform-Android-3DDC84?logo=android&logoColor=white)](https://developer.android.com/)
[![Framework](https://img.shields.io/badge/Flutter-Dart-02569B?logo=flutter&logoColor=white)](https://flutter.dev/)
[![Backend](https://img.shields.io/badge/Firebase-Firestore%20%7C%20Auth%20%7C%20Functions-FFCA28?logo=firebase&logoColor=black)](https://firebase.google.com/)
[![Security](https://img.shields.io/badge/Focus-Application%20Security-FF0000?logo=owasp&logoColor=white)](https://owasp.org/)
[![License](https://img.shields.io/badge/License-Private-lightgrey)]()

---

**⚠️ This is a showcase repository.** The source code is hosted in a private repo to protect production credentials and business data. What you'll find here is the full technical documentation, architecture decisions, security model, and project structure — enough to understand every engineering choice I made.

---

## Table of Contents

- [Context](#context)
- [Security Architecture](#security-architecture)
  - [Authentication Flow](#authentication-flow)
  - [Authorization — RBAC via JWT Custom Claims](#authorization--rbac-via-jwt-custom-claims)
  - [Server-Side Validation (Firestore Security Rules)](#server-side-validation-firestore-security-rules)
  - [Audit Trail](#audit-trail)
- [System Architecture](#system-architecture)
- [Offline-First Data Layer](#offline-first-data-layer)
- [Features](#features)
- [Data Model (NoSQL)](#data-model-nosql)
- [UX Considerations](#ux-considerations)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Current Status](#current-status)
- [Versão em Português (pt-BR)](#versão-em-português-pt-br)

---

## Context

My father-in-law runs a water distribution company out of a small town in Pernambuco. He's 75. His "system" was an Excel workbook with one sheet per month — driver names across columns, trip values down the rows, manual SUM formulas at the bottom. It worked for years, but it had obvious problems: the file lived on a single laptop, there was no access control, anyone who opened it could see (or change) everything, and end-of-month reporting took him an entire afternoon.

The drivers had it worse. They operate across rural highways where cell signal drops to nothing for hours at a time. At the end of each shift they'd report completed deliveries on paper or over the phone. My father-in-law would then manually key everything into Excel. Data loss, transcription errors, duplicate entries — all of it happened regularly.

I built this application to solve those problems, and I deliberately framed it as a security engineering exercise. I'm transitioning into cybersecurity, and I wanted a project where security decisions weren't abstract — they had to work for real people, on real infrastructure, with real constraints.

Every design choice in this system — from how authentication works to why I wrote custom sync logic instead of relying on Firestore's built-in cache — came from an actual operational need, not a tutorial.

---

## Security Architecture

This is the core of the project. I followed defense-in-depth principles: no single layer is solely responsible for security. Client-side controls exist for UX; server-side controls exist for actual enforcement. Nothing trusts the client.

### Authentication Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          FIRST LOGIN                                     │
│                                                                          │
│  [Email + Password] ──→ Firebase Auth ──→ JWT issued ──→ Session cached  │
│                                                ↓                         │
│                                         [Prompt PIN setup]               │
│                                         [Offer biometric enrollment]     │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                        RETURNING USER                                    │
│                                                                          │
│  [Biometric scan] ──→ local_auth ──→ Unlock cached session               │
│         │ (fallback)                                                     │
│         ↓                                                                │
│  [PIN entry] ──→ Android Keystore validation ──→ Unlock cached session   │
│         │ (3 failures → 30s lockout)                                     │
│         ↓ (10 total failures)                                            │
│  [Session wipe] ──→ Force re-authentication with email/password          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Design rationale:**

The intention was to mirror banking app patterns. The Firebase Auth token persists locally after the first login. Biometrics and the PIN don't re-authenticate against Firebase — they just gate access to the already-stored token. This eliminates password fatigue for field workers who open and close the app many times during a shift.

The PIN is stored in the **Android Keystore** via `flutter_secure_storage` with `encryptedSharedPreferences: true`. It never touches the network. It never gets written to Firestore. It's a local-only unlock mechanism backed by hardware-level encryption.

**Brute-force mitigation** is handled entirely on-device:
- After **3 consecutive wrong PINs** → 30-second cooldown (UI shows countdown timer)
- After **10 cumulative failures** → full session wipe, PIN is cleared, user must re-authenticate with email/password from scratch

I intentionally chose not to add server-side rate limiting for the PIN because the PIN never goes to the server. The rate limiting lives where the credential lives — on the device.

### Authorization — RBAC via JWT Custom Claims

Roles are injected into the Firebase Auth JWT via a **Cloud Function** (`setUserRole`). Not stored in a Firestore document that the client queries — baked into the token itself.

```
Cloud Function (admin-only callable)
         │
         ↓
  admin.auth().setCustomUserClaims(uid, { role: 'admin' | 'secretary' })
         │
         ↓
  JWT token now carries: { ..., "role": "admin" }
         │
         ├──→ Client: GoRouter guards check token.role before allowing navigation
         │        ↳ Secretary can't navigate to financial dashboard
         │        ↳ Enforced at the routing layer, not just hidden buttons
         │
         └──→ Server: Firestore Security Rules inspect request.auth.token.role
                  ↳ Can't be bypassed by direct REST/gRPC calls to Firestore
```

**Why Custom Claims instead of a `roles` collection?**

A document lookup adds latency and an extra Firestore read on every rule evaluation. Custom Claims are already present in the JWT that accompanies every request — zero overhead, zero additional reads. The tradeoff is that role changes require a token refresh, but in this system roles change maybe once a year, so that's a non-issue.

### Server-Side Validation (Firestore Security Rules)

The client has validation for user experience (showing error messages, preventing empty fields). But security validation happens on the server. A motivated attacker could bypass the Flutter app entirely and hit Firestore's REST API directly — the Security Rules are what actually stop them.

What the rules enforce:

| Rule | Example |
|---|---|
| **Required fields** | `request.resource.data.keys().hasAll(['driver_name', 'truck_number', ...])` |
| **Type enforcement** | `delivery_value_brl is number` |
| **Range validation** | `delivery_value_brl > 0 && delivery_value_brl < 100000` |
| **Enum constraints** | `payment_status in ['pending', 'paid']` |
| **Ownership verification** | `request.auth.uid == resource.data.created_by` |
| **Write restrictions** | `monthly_summaries` collection: `allow write: if false` (Cloud Functions only) |
| **Soft deletes** | `users` collection: `allow delete: if false` (deactivate via `is_active` flag instead) |

The `monthly_summaries` collection deserves special mention: it's written exclusively by Cloud Functions using the Admin SDK. Client-side writes are blocked entirely (`allow write: if false`). This prevents any client from manipulating aggregated financial data — the aggregation runs server-side inside a Firestore Transaction to guarantee consistency.

### Audit Trail

Every trip record includes:

| Field | Source | Purpose |
|---|---|---|
| `created_at` | Server timestamp, set once, immutable | When the record was created |
| `created_by` | `request.auth.uid` | Who created it |
| `device_id` | `device_info_plus` (runtime) | Which physical device was used |
| `synced_at` | Set on successful cloud write | Confirms the record reached Firestore |

This gives full traceability per record: who, when, from which device, and whether the sync completed. In a security incident or a financial dispute, every record has a chain of evidence back to a specific authenticated user on a specific device.

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         FLUTTER MOBILE APP                               │
│                                                                          │
│   ┌──────────┐    ┌─────────────┐    ┌───────────────────────────────┐   │
│   │   Auth    │    │  Trips &    │    │   Dashboard & Reports        │   │
│   │ Feature   │    │  Fleet      │    │   Feature                    │   │
│   └─────┬────┘    └──────┬──────┘    └──────────────┬────────────────┘   │
│         │                │                          │                    │
│   ┌─────┴────────────────┴──────────────────────────┴────────────────┐   │
│   │                  Riverpod  (State Management)                    │   │
│   └─────┬────────────────┬──────────────────────────┬────────────────┘   │
│         │                │                          │                    │
│   ┌─────┴────────────────┴──────────────────────────┴────────────────┐   │
│   │              Repository Layer (Domain Interfaces)                │   │
│   └─────┬────────────────┬───────────────────────────────────────────┘   │
│         │                │                                               │
│   ┌─────┴─────┐   ┌─────┴──────┐                                         │
│   │   Drift   │   │   Sync     │                                         │
│   │ (SQLite)  │◄─►│  Engine    │                                         │
│   └───────────┘   └─────┬──────┘                                         │
│                         │                                                │
└─────────────────────────┼────────────────────────────────────────────────┘
                          │ HTTPS / TLS 1.3
┌─────────────────────────┼────────────────────────────────────────────────┐
│                    FIREBASE  (Backend)                                   │
│                                                                          │
│   ┌────────────┐  ┌─────┴──────┐  ┌─────────────────────────────────┐    │
│   │  Firebase  │  │   Cloud    │  │      Cloud Functions (v2)       │    │
│   │   Auth     │  │  Firestore │  │  ┌─ setUserRole (RBAC claims)   │    │
│   │            │  │            │  │  ├─ aggregateTrips (summaries)  │    │
│   │            │  │            │  │  └─ aggregateExpenses           │    │
│   └────────────┘  └────────────┘  └─────────────────────────────────┘    │ 
│                                                                          │
│   ┌──────────────────────────────────────────────────────────────────┐   │
│   │             Firestore Security Rules                             │   │
│   │    RBAC + Schema Validation + Immutability + Ownership Checks    │   │
│   └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

### Architectural Decisions

| Decision | Reasoning |
|---|---|
| **Riverpod** over BLoC/Provider | Compile-time safety on dependency injection. No `BuildContext` required for state access. Simpler testability story. |
| **GoRouter** for navigation | Declarative route definitions. The redirect logic handles RBAC guards cleanly — a secretary attempting to navigate to `/dashboard` gets redirected before the page even builds. |
| **Feature-based folder structure** | Each domain (auth, trips, fleet, expenses, dashboard, reports) is self-contained with its own `data/`, `domain/`, and `presentation/` layers. No cross-feature imports except through shared abstractions. |
| **Repository pattern** | The domain layer defines interfaces. The data layer provides implementations. Swapping SQLite for Firestore (or mocking for tests) requires changing one provider, not rewriting business logic. |
| **Drift** over raw `sqflite` | Type-safe SQL queries at compile time. Reactive streams for UI binding. Built-in migration support for schema evolution without data loss. |
| **Cloud Functions v2** | Needed for two things only: injecting Custom Claims (can't be done client-side — that would be a privilege escalation vulnerability) and pre-computing monthly summaries inside Firestore Transactions. |

---

## Offline-First Data Layer

Drivers spend most of their shift in areas with no cell signal. Any architecture that depends on network availability would be broken by design.

### Why Not Firestore's Built-in Offline Cache?

Firestore has an offline persistence mode. I evaluated it and chose not to rely on it for several reasons:

1. **No guaranteed persistence after process kill** — some Android OEMs aggressively kill background processes, and Firebase's cache lives in memory until flushed
2. **No visibility into pending operations** — the user has no way to know what's synced and what isn't
3. **Can't run complex local queries** — the offline cache is a transparent layer, not a proper local database
4. **No control over retry behavior** — exponential backoff, queue ordering, battery-aware abort are all things I needed to control

### How It Works

```
[User Input] → [Drift/SQLite] → [Sync Queue] → [Sync Engine] → [Firestore]
    (UI)       (source of truth    (persistent     (backoff +      (cloud
                on device)          retry queue)    network-aware)   authority)
```

| Component | Technology | Responsibility |
|---|---|---|
| **Local DB** | Drift (SQLite wrapper) | All writes land here first. The UI always reads from local. Instant response times regardless of network. |
| **Sync Queue** | SQLite table (`sync_queue`) | Persistent queue of pending operations. Survives app kills. Each entry tracks: collection, document ID, operation type, attempt count, next retry timestamp. |
| **Sync Engine** | Custom Dart service | Processes the queue with exponential backoff (1s → 2s → 4s → ... → 60s cap). On `FirebaseException` with code `unavailable` or `network-request-failed`, it aborts the entire queue immediately — no point burning battery retrying when there's no signal. |
| **Background Sync** | WorkManager | Continues processing the queue even when the app is in the background or the process has been killed by the OS. |
| **Network Detection** | connectivity_plus | Triggers an immediate sync attempt when connectivity is restored. |

**User-facing status (always visible in the AppBar):**
- ✅ Synced — all records confirmed in Firestore
- 🔄 Pending (N) — N records in the queue waiting for network
- ❌ Offline — no connection, but all data is safe locally

---

## Features

### Fleet Operations
- **Truck registry** — number, capacity (liters), active/inactive status
- **Driver registry** — name, active status, soft-delete only (never hard-delete operational data)
- **Dynamic assignment** — any driver can be assigned to any truck on a per-trip basis; the fleet changes constantly

### Trip Registration
- **4-step wizard flow**: driver → truck → client → delivery value
- **Receipt-style confirmation screen** before saving — shows all entered data in a summary card
- **Instant local persistence** — the record exists in SQLite the moment the user confirms, sync happens asynchronously
- **Haptic feedback** on successful save — the driver knows it worked without reading the screen

### Client Management
- Client database with search and quick-add from within the trip wizard (no need to navigate away)

### Expense Tracking
- Categories: fuel, maintenance, salary, other
- Monthly expense breakdown on the admin dashboard
- **Fuel cost percentage** calculated as fuel expenses ÷ total revenue (not ÷ total expenses) — this specific metric is what the business owner uses to evaluate route profitability

### Admin Dashboard
- **KPI grid**: total revenue, trip count, average per trip, fuel cost %, pending payments, net profit
- **Period filters**: today / this week / this month, with month-by-month navigation
- **Driver breakdown table** — mirrors the column layout from the owner's original Excel spreadsheet
- **Truck revenue bar chart** — revenue per vehicle
- **6-month trend line chart** — revenue evolution for seasonal pattern analysis
- **Interactive charts** via `fl_chart` — tap for values, smooth animations on filter changes

### PDF Reports
- One-tap generation from any filtered view
- Report includes: KPI summary, driver table, truck table, full trip list
- Share via WhatsApp or print — the business owner prints a copy every month for his physical binder

---

## Data Model (NoSQL)

Denormalized schema optimized for read performance and minimal Firestore cost on the dashboard.

```
users/                             trips/
├── uid  (Firebase Auth PK)        ├── id  (UUID, generated client-side)
├── name                           ├── driver_name
├── role  (admin | secretary)      ├── truck_number
├── created_at                     ├── truck_capacity_liters
└── is_active                      ├── client_name
                                   ├── delivery_value_brl  (double)
clients/                           ├── payment_status  (pending | paid)
├── id  (UUID)                     ├── date  (YYYY-MM-DD)
├── name                           ├── month_year  (YYYY-MM, partition key)
└── created_at                     ├── created_at  (immutable server timestamp)
                                   ├── created_by  (auth UID)
expenses/                          ├── device_id  (audit trail)
├── id  (UUID)                     └── synced_at
├── month_year  (YYYY-MM)
├── category  (fuel | maintenance | salary | other)
├── description
├── value_brl  (double)
├── created_at
└── created_by  (admin UID)

monthly_summaries/                 (written exclusively by Cloud Functions)
├── id  (YYYY-MM)
├── total_revenue_brl
├── total_trips
├── total_expenses_brl
├── profit_brl
├── avg_revenue_per_trip
├── fuel_cost_percentage
├── pending_payments_count / value
├── driver_breakdown  (map)
├── truck_breakdown  (map)
└── last_updated
```

`month_year` acts as a partition key. Dashboard queries always filter by this field first, which keeps Firestore reads fast and costs predictable. The `monthly_summaries` collection is pre-computed by Cloud Functions running inside Firestore Transactions — the client never runs aggregation queries.

---

## UX Considerations

The primary user is a 75-year-old business owner who has never used a computer application more complex than Excel. Every UX decision was filtered through the question: *"will Nelson understand this without me explaining it?"*

| Constraint | Solution |
|---|---|
| Limited tech literacy | High-contrast color palette, large typography (data values ≥ 20sp, labels ≥ 12sp), minimal navigation depth |
| Excel familiarity | PDF reports mirror the exact tabular layout of his old spreadsheets |
| Low patience for errors | 4-step wizard with one question per screen, receipt confirmation before any save |
| WhatsApp as primary communication | PDF share button integrates directly with WhatsApp share sheet |
| "Did it work?" anxiety | Full-screen success state with haptic feedback after trip creation |
| "Is my data safe?" | Persistent sync status indicator in the AppBar — never hidden, always visible |

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| **UI Framework** | Flutter 3.x / Dart | Native Android performance, single codebase, strong typing |
| **State Management** | Riverpod 2.x | Compile-safe dependency injection, reactive, testable |
| **Navigation** | GoRouter | Declarative routing with redirect-based RBAC guards |
| **Local Database** | Drift (SQLite ORM) | Type-safe queries, reactive streams, migration support |
| **Backend** | Firebase (Auth, Firestore, Cloud Functions v2) | Managed infrastructure, Security Rules as access control layer |
| **Visualization** | fl_chart | Interactive charts with animation support |
| **PDF** | pdf + printing | Native PDF generation and OS-level share integration |
| **Local Security** | flutter_secure_storage + local_auth | PIN in Android Keystore + biometric authentication |
| **Background Processing** | WorkManager | OS-managed background task execution for offline sync |
| **Networking** | connectivity_plus | Real-time network state monitoring |
| **Code Generation** | freezed, json_serializable, drift_dev | Immutable models, type-safe serialization, zero boilerplate |

---

## Project Structure

```
lib/
├── main.dart                             # Entry point, Firebase init
├── app.dart                              # MaterialApp config, theme, router binding
├── core/
│   ├── providers/                        # Riverpod DI — database, sync, auth providers
│   ├── routing/
│   │   └── app_router.dart               # GoRouter config with RBAC redirect guards
│   ├── theme/                            # Color system, typography scale
│   └── services/
│       ├── local_db/                     # Drift schema, tables, migrations
│       ├── sync_service.dart             # Sync engine — queue processing, backoff, abort
│       ├── initial_sync_service.dart     # First-launch pull from Firestore → local DB
│       └── background_sync_setup.dart    # WorkManager registration
├── features/
│   ├── auth/
│   │   ├── data/                         # FirebaseAuthRepository, PinService, BiometricService
│   │   ├── domain/                       # Auth state models, session abstractions
│   │   └── presentation/                 # LoginScreen, PinScreen, PinSetupScreen
│   ├── trips/
│   │   ├── data/                         # DriftTripRepository (local), Firestore sync
│   │   ├── domain/                       # Trip model, repository interface
│   │   └── presentation/                 # Trip wizard (4-step), trip list, detail view
│   ├── fleet/                            # Truck & driver management (same layer pattern)
│   ├── clients/                          # Client CRUD with inline creation from wizard
│   ├── expenses/                         # Categorized expense tracking
│   ├── payments/                         # Payment status toggling (pending → paid)
│   ├── dashboard/                        # KPI grid, charts, period filters, data providers
│   └── reports/                          # PDF builder, print/share integration
└── shared/
    ├── widgets/                          # KpiCard, SyncStatusIndicator, reusable components
    └── screens/                          # Management hub, shared layout shells

functions/
├── index.js                              # Cloud Functions: setUserRole, aggregateTrips/Expenses
└── package.json

firestore.rules                           # Security Rules: RBAC + schema validation + immutability
```

Each feature follows the `data/` → `domain/` → `presentation/` layered pattern consistently.

---

## Current Status

The application is deployed and in daily use by the business.

- ✅ Multi-factor local authentication (Firebase Auth + biometrics + PIN with brute-force protection)
- ✅ Role-based access control with server-side enforcement via JWT Custom Claims
- ✅ Full fleet management (trucks, drivers, clients) with soft-delete policy
- ✅ Trip registration wizard with offline-first persistence
- ✅ Categorized expense tracking with monthly aggregation
- ✅ Financial dashboard with KPI grid, interactive charts, and period filtering
- ✅ Custom sync engine with exponential backoff and battery-aware abort
- ✅ Background sync via WorkManager (survives process kill)
- ✅ PDF report export with WhatsApp and printer sharing
- ✅ Audit trail on every record (who, when, from which device, sync status)
- ✅ Firestore Security Rules with schema validation, type checking, and range boundaries

---

## Get in Touch

If you're reading this as part of a hiring process or because you're curious about the implementation details — I'm happy to walk through the code in a technical interview, discuss the security model in depth, or do a live demo of the running application.

This project represents about where I am right now: building things that actually work for real people, with security baked in from day one rather than bolted on after the fact.

→ [GitHub](https://github.com/nicokaka) · [LinkedIn](https://linkedin.com/in/)

---
---

# 🇧🇷 Versão em Português (pt-BR)

## Contexto

Meu sogro tem uma distribuidora de água no interior de Pernambuco. Ele tem 75 anos e o "sistema" dele era uma planilha Excel com uma aba por mês — nomes dos motoristas nas colunas, valores das viagens nas linhas, fórmulas de SOMA no final. Funcionava, mas tinha problemas óbvios: o arquivo ficava num notebook só, qualquer pessoa que abria podia ver e alterar tudo, e o fechamento de mês levava uma tarde inteira.

Os motoristas tinham a situação pior. Eles rodam em estradas rurais onde o sinal de celular some por horas. No final do turno, reportavam as entregas no papel ou por telefone. Meu sogro digitava tudo manualmente no Excel. Perda de dados, erros de digitação, entradas duplicadas — tudo isso acontecia com frequência.

Eu construí esse aplicativo para resolver esses problemas, e intencionalmente tratei como um exercício de engenharia de segurança. Estou migrando para cybersecurity e queria um projeto onde as decisões de segurança não fossem abstratas — precisavam funcionar para pessoas reais, em infraestrutura real, com restrições reais.

Cada decisão de design nesse sistema — desde como a autenticação funciona até por que escrevi lógica de sync customizada ao invés de depender do cache offline nativo do Firestore — veio de uma necessidade operacional concreta, não de um tutorial.

---

## Arquitetura de Segurança

### Autenticação

- **Primeiro acesso:** Email/senha via Firebase Auth → token JWT emitido → sessão cacheada no dispositivo
- **Acessos seguintes:** Biometria (digital/face) como método primário → PIN no Android Keystore como fallback
- O PIN **nunca** sai do dispositivo — é um mecanismo de unlock local, armazenado com criptografia de hardware
- **Proteção contra brute-force:** 3 tentativas erradas → cooldown de 30s → após 10 falhas totais, sessão é apagada e o usuário precisa autenticar do zero

### Autorização — RBAC via Custom Claims no JWT

- Roles (`admin` | `secretary`) são injetadas no JWT via Cloud Function — não ficam num documento Firestore que o cliente consulta
- **Client-side:** GoRouter impede navegação para rotas restritas (secretária não acessa dashboard financeiro)
- **Server-side:** Firestore Security Rules verificam `request.auth.token.role` em toda operação — não dá pra burlar chamando a API REST diretamente

### Validação Server-Side (Firestore Security Rules)

Validação no cliente é para UX. Validação de segurança vive no servidor. As Security Rules garantem:

- Campos obrigatórios (`keys().hasAll([...])`)
- Tipagem (`delivery_value_brl is number`)
- Limites de valor (`> 0 && < 100000`)
- Ownership (`request.auth.uid == resource.data.created_by`)
- Coleção `monthly_summaries` com `allow write: if false` — somente Cloud Functions escrevem

### Trilha de Auditoria

Cada registro de viagem carrega: `created_at` (timestamp do servidor, imutável), `created_by` (UID do Firebase Auth), `device_id` (dispositivo físico via `device_info_plus`) e `synced_at` (confirmação de sync).

---

## Funcionalidades

- **Gestão de Frota:** Cadastro de caminhões e motoristas, atribuição flexível por viagem, soft-delete
- **Registro de Viagens:** Wizard de 4 etapas com confirmação tipo recibo — motorista → caminhão → cliente → valor
- **Controle de Despesas:** Combustível, manutenção, salários e outros, com breakdown mensal
- **Painel Financeiro:** KPIs (faturamento, viagens, média por viagem, % combustível, pendentes, lucro líquido), gráficos interativos, filtros por período
- **Relatório PDF:** Exportação com um toque, formato tabular que o dono já conhece do Excel, compartilhamento via WhatsApp
- **Funciona Offline:** SQLite local como fonte de verdade, fila de sync persistente com backoff exponencial, sync em background via WorkManager

## Stack

Flutter/Dart · Firebase (Firestore, Auth, Cloud Functions v2) · Riverpod · Drift (SQLite) · GoRouter · fl_chart · PDF nativo · flutter_secure_storage + local_auth · WorkManager · connectivity_plus

---

<p align="center">
  <sub>Projetado, arquitetado e desenvolvido por <a href="https://github.com/nicokaka">Nicolas</a>.</sub>
</p>

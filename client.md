# Client Management – Implementation Plan

> Service: `bcd-japan-tenant-policy-service` (NestJS 11)
> Architecture: **DDD + Clean Architecture** 
> Stack decisions (confirmed): **TypeORM + PostgreSQL**, domain events defined but **broker deferred**,
> scope = **Onboarding + Registry + Approval** only, designed so profile editing /
> relationships / authority can be added later without restructuring.
> Source material: legacy BTOS `Client Management – Module Deep-Dive`, F1 Backend Services UI (6-step wizard + registry screens).

---

## 1. Goal & non-goals

### 1.1 Goal

Build the **Client Management** bounded context that backs:

- The **Client Registry** dashboard (counts, search, three tabs: Client Registry / In Approval / Rejected).
- The **6-step "Add new client" onboarding wizard** (Basic → Company → Conditions of Use → Bank & Billing → Service Fees → Review).
- The **approval lifecycle** (submit → in-approval → approved/rejected) and the parallel **finance-setup** sub-state.
- **Logical deactivation** (no hard delete) and **duplicate-company-name blocking on create**.

### 1.2 In scope

| Area | Included |
|---|---|
| Client onboarding draft + 6-step data capture | ✅ |
| Registry list / search / pagination / tab filtering | ✅ |
| Dashboard counters (clients tracked, pending approvals, rejected) | ✅ |
| Submit-for-approval, approve, reject (with reason) | ✅ |
| Finance-setup status transitions | ✅ |
| Logical deactivation (soft delete) | ✅ |
| Duplicate company-name guard on create | ✅ |
| Domain events (defined, published to an in-process dispatcher; broker adapter stubbed) | ✅ |
| Excel/CSV "import from template" prefill endpoint | ✅ (parse + map only; no bulk-create) |

### 1.3 Out of scope (designed for, not built)

Profile editing (passport/visa/card history), relationships (代行 agents / 同行 companions / arrangers), approval-route engine, authority/`RecAuthrity` model, SSO/supplier credentials, bulk profile upload, traveler/program/ticketing/finance/visa modules. Each is a **future module or future aggregate** under the same layering — see §13 Extensibility.

### 1.4 Module split & bounded-context map

Client Management spans **three sibling bounded contexts** — separate modules under `src/modules/`, each with full `{domain,application,infrastructure,presentation}` layers. This plan details **`clients`**; `service-fees` and `client-reports` are peer modules, specified here at **contract level** and built alongside.

| Module | Owns (system of record) | Does NOT own |
|---|---|---|
| **`clients`** | Client lifecycle (`DRAFT→ACTIVE→DEACTIVATED`), onboarding wizard data for steps 1–4 & 6, approval/finance status, credit limit, registry & counters | Any fee data; report definitions |
| **`service-fees`** | **All fee data** — schedule definitions, the onboarding (Step 5) schedule, per-case overrides (`ServiceFeeByCase`), fee change-request approval workflow (`ServiceFeeRequests`) with its own request state machine, approvers & audit | Client lifecycle/status |
| **`client-reports`** | Report definitions, schedules, run history | Client or fee data (read-only consumer) |

**Cross-module rules**

- Reference other contexts by `ClientId` only. No cross-module foreign keys, no shared ORM entities, no importing another module's `domain/`.
- Collaboration via application **ports** (interfaces in the calling module's `application/ports`, adapters in its `infrastructure`).

**Fee ownership — decided: Option A.** `service-fees` is the single source of truth for *all* fee data, including the onboarding schedule. The `Client` aggregate holds **no fee VO and no fee column**. The Step 5 endpoint is exposed on the clients onboarding controller for UI convenience but delegates through `ServiceFeePort` to a `service-fees` use case. Submission precondition "fees configured" is checked by querying the port — never by reading client-owned data.

**Onboarding consistency.** As a modular monolith on one Postgres datasource, the Step 5 write (in `service-fees`) and the client draft update (in `clients`) run in a **shared local transaction / unit-of-work** spanning both modules' repositories — no distributed saga yet. The port boundary is preserved so that when `service-fees` is later extracted to its own service, the call becomes an **outbox-published command + saga** (compensation on `ClientSubmittedForApproval` failure) with zero domain change.

**`client-reports` contract.** Subscribes to `ClientApproved` / `ClientDeactivated` (once the broker lands) or reads the `clients` read model by `ClientId`. Owns only definitions/schedules/runs; never writes client or fee data.

---

## 2. Architecture & dependency rule

Clean Architecture concentric layers. **The dependency rule: source code dependencies point inward only.**

```
presentation ──▶ application ──▶ domain ◀── infrastructure
                      │                          │
                      └────────── (DI: app depends on domain interfaces,
                                   infrastructure implements them)
```

- **domain**: pure TypeScript. No NestJS decorators, no TypeORM, no DTOs, no HTTP. Knows nothing about the outside.
- **application**: orchestration (use cases). Depends on domain only. Defines/consumes repository & port **interfaces** (in domain). NestJS `@Injectable()` allowed here (DI wiring only).
- **infrastructure**: implements domain repository interfaces with TypeORM; messaging adapters; config. Depends on domain + application.
- **presentation**: controllers/guards/interceptors. Depends on application use cases only — never on domain entities directly (maps via DTOs).

Enforcement: ESLint `no-restricted-imports` (or `eslint-plugin-boundaries`) rules added so `domain/**` cannot import from `application|infrastructure|presentation|@nestjs/*|typeorm`.

---

## 3. Target folder structure

Module-per-bounded-context; layers inside each module; cross-cutting in `shared/` and `config/`. Tree below is the **`clients`** module in full; `modules/service-fees/` and `modules/client-reports/` are peer trees with the same layer layout (not expanded here — see §1.4 for their contracts).

```
src/
├── main.ts
├── app.module.ts
│
├── config/
│   ├── app.config.ts                 # port, env, global prefix
│   ├── database.config.ts            # TypeORM datasource (Postgres)
│   └── messaging.config.ts           # broker settings (placeholder, not wired)
│
├── shared/
│   ├── domain/
│   │   ├── aggregate-root.base.ts    # records & exposes domain events
│   │   ├── entity.base.ts
│   │   ├── value-object.base.ts
│   │   ├── domain-event.interface.ts
│   │   └── result.ts                 # Result/Either for typed domain failures
│   ├── application/
│   │   ├── use-case.interface.ts
│   │   └── pagination.dto.ts
│   ├── infrastructure/
│   │   └── typeorm/base.orm-entity.ts  # id, createdAt, updatedAt, deletedAt
│   ├── domain-events/
│   │   ├── domain-event-dispatcher.ts          # in-process dispatcher (interface)
│   │   └── in-memory-event-dispatcher.ts       # default impl (broker deferred)
│   ├── constants/
│   ├── types/
│   └── utils/
│
└── modules/
    └── client-management/
        ├── client-management.module.ts        # NestJS wiring + DI tokens
        │
        ├── presentation/
        │   ├── controllers/
        │   │   ├── client-registry.controller.ts     # list/search/counters
        │   │   ├── client-onboarding.controller.ts    # 6-step draft + submit
        │   │   ├── client-approval.controller.ts      # approve/reject
        │   │   └── client-template.controller.ts      # Excel/CSV prefill
        │   ├── dto/                                    # request/response DTOs (class-validator)
        │   │   ├── requests/
        │   │   │   ├── create-client-draft.request.ts
        │   │   │   ├── update-basic-details.request.ts
        │   │   │   ├── update-company-details.request.ts
        │   │   │   ├── update-conditions-of-use.request.ts
        │   │   │   ├── update-bank-billing.request.ts
        │   │   │   ├── update-service-fees.request.ts
        │   │   │   ├── submit-client.request.ts
        │   │   │   ├── reject-client.request.ts
        │   │   │   └── list-clients.query.ts
        │   │   └── responses/
        │   │       ├── client-summary.response.ts      # registry row
        │   │       ├── client-detail.response.ts        # review screen
        │   │       └── registry-counters.response.ts
        │   ├── guards/                                  # placeholder: AuthGuard, ScopeGuard
        │   ├── interceptors/                            # response envelope, logging
        │   └── filters/
        │       └── domain-exception.filter.ts           # maps domain errors → HTTP
        │
        ├── application/
        │   ├── use-cases/
        │   │   ├── create-client-draft.use-case.ts
        │   │   ├── update-client-section.use-case.ts    # generic per-step persist
        │   │   ├── submit-client-for-approval.use-case.ts
        │   │   ├── approve-client.use-case.ts
        │   │   ├── reject-client.use-case.ts
        │   │   ├── deactivate-client.use-case.ts
        │   │   ├── update-finance-setup-status.use-case.ts
        │   │   ├── list-clients.use-case.ts
        │   │   ├── get-client-detail.use-case.ts
        │   │   ├── get-registry-counters.use-case.ts
        │   │   └── parse-client-template.use-case.ts
        │   ├── dto/                                      # application-layer commands/queries
        │   ├── mappers/
        │   │   ├── client.app-mapper.ts                  # domain ↔ response DTO
        │   │   └── client-section.assembler.ts           # wizard step ↔ VO
        │   ├── ports/
        │   │   ├── template-parser.port.ts               # interface for xlsx/csv parsing
        │   │   └── service-fee.port.ts                    # → service-fees module (Step 5 delegate + "configured?" query)
        │   └── services/
        │       └── client-onboarding.orchestrator.ts     # coordinates multi-step flow
        │
        ├── domain/
        │   ├── aggregates/
        │   │   └── client.aggregate.ts                   # root: lifecycle + invariants
        │   ├── entities/
        │   │   └── (future: passport, visa, card sub-entities)
        │   ├── value-objects/
        │   │   ├── client-id.vo.ts
        │   │   ├── lcn.vo.ts                              # Legal Client Number
        │   │   ├── company-code.vo.ts
        │   │   ├── onboarding-status.vo.ts                # state machine
        │   │   ├── finance-setup-status.vo.ts
        │   │   ├── person-name.vo.ts
        │   │   ├── passport-info.vo.ts
        │   │   ├── visa-info.vo.ts
        │   │   ├── company-profile.vo.ts                  # legal entity + flags
        │   │   ├── travel-preferences.vo.ts               # conditions of use
        │   │   ├── service-policy.vo.ts
        │   │   ├── banking-details.vo.ts
        │   │   └── rejection-reason.vo.ts                 # (no fee VO here — fees owned by service-fees module, §1.4)
        │   ├── repositories/
        │   │   └── client.repository.ts                   # interface ONLY
        │   ├── services/
        │   │   ├── company-name-uniqueness.service.ts     # domain service interface
        │   │   └── lcn-generator.service.ts               # domain service interface
        │   ├── events/
        │   │   ├── client-draft-created.event.ts
        │   │   ├── client-submitted-for-approval.event.ts
        │   │   ├── client-approved.event.ts
        │   │   ├── client-rejected.event.ts
        │   │   ├── client-finance-status-changed.event.ts
        │   │   └── client-deactivated.event.ts
        │   └── exceptions/
        │       ├── duplicate-company-name.exception.ts
        │       ├── invalid-status-transition.exception.ts
        │       ├── client-not-found.exception.ts
        │       └── client-validation.exception.ts
        │
        └── infrastructure/
            ├── database/
            │   ├── entities/
            │   │   └── client.orm-entity.ts               # TypeORM table mapping
            │   ├── migrations/
            │   │   └── <ts>-create-client.ts
            │   ├── repositories/
            │   │   └── client.repository.impl.ts           # implements domain interface
            │   ├── mappers/
            │   │   └── client.orm-mapper.ts                # ORM row ↔ aggregate
            │   └── services/
            │       ├── company-name-uniqueness.impl.ts
            │       └── lcn-generator.impl.ts
            ├── messaging/
            │   └── producers/
            │       └── client-event.producer.ts            # stub: logs/queues, broker TODO
            └── template/
                └── xlsx-template-parser.ts                 # TemplateParserPort impl
```

---

## 4. Domain model

### 4.1 Aggregate: `Client`

**Aggregate root.** Identity = `ClientId` (UUID). One aggregate = one onboarded client/company profile (registry row). For the current scope, company + traveller-basic data live together on the root as value objects; when profile editing lands, passport/visa/cards split into child entities under the same root (see §13).

Root state (all VOs, immutable; aggregate mutates by replacing VOs):

| Field | VO / type | Source UI step |
|---|---|---|
| `id` | `ClientId` | system |
| `lcn` | `Lcn` | system-generated on draft create |
| `companyCode` | `CompanyCode` | derived from company name/manual |
| `basicDetails` | `{ personName, title, gender, dob, profession, passport, visa }` | Step 1 |
| `companyProfile` | `CompanyProfile` (names EN/Kanji, addresses, contact, tax id, industry, category, vip, handleWithCare, internalReference, country) | Step 2 |
| `travelPreferences` | `TravelPreferences` | Step 3 |
| `servicePolicy` | `ServicePolicy` | Step 3 |
| `bankingDetails` | `BankingDetails` | Step 4 |
| _(no fee field)_ | — owned by **service-fees** module; Step 5 delegated via `ServiceFeePort` | Step 5 |
| `onboardingStatus` | `OnboardingStatus` | lifecycle |
| `financeSetupStatus` | `FinanceSetupStatus` | lifecycle |
| `rejectionReason` | `RejectionReason \| null` | approval |
| `active` | boolean (soft-delete flag) | deactivate |
| audit | `createdAt/By`, `updatedAt/By`, `deactivatedAt` | system |

**Factory:** `Client.createDraft(input, lcnGenerator, nameUniqueness): Result<Client>`
- Generates `Lcn` via injected `LcnGeneratorService`.
- Calls `CompanyNameUniquenessService` → throws `DuplicateCompanyNameException` if taken.
- Initial state: `onboardingStatus = DRAFT`, `financeSetupStatus = NOT_STARTED`, `active = true`.
- Emits `ClientDraftCreatedEvent`.

**Section mutators** (one per wizard step) — only allowed while `DRAFT` or (later) `REJECTED` being revised:
`updateBasicDetails`, `updateCompanyProfile`, `updateConditionsOfUse`, `updateBankingDetails`. Each validates the VO and re-evaluates completeness. **Step 5 has no aggregate mutator** — the service-fees step is delegated to the `service-fees` module via `ServiceFeePort`; the aggregate only tracks *whether* a schedule exists (queried via the port at `isReadyForSubmission()`).

**Lifecycle methods** with guarded transitions (see §5):
`submitForApproval()`, `approve(approver)`, `reject(approver, reason)`, `deactivate(actor)`, `setFinanceSetupStatus(next)`.

**Invariants enforced inside the aggregate**

1. Company name (EN, normalized) is unique across non-deactivated clients — checked via domain service at create (and at company-name change).
2. No hard delete: `deactivate()` only flips `active=false` + status `DEACTIVATED`; repository never exposes a `delete`.
3. Status transitions only along the legal state machine; illegal transition → `InvalidStatusTransitionException`.
4. `submitForApproval()` requires all six steps "complete" per `isReadyForSubmission()` (required-field set defined per VO; mirrors legacy `RecClient.IsRequiredInput`, but rules live in VOs/domain service, not in a god-object).
5. `reject()` requires a non-empty `RejectionReason`.
6. Finance-setup status is independent of onboarding status but `ACTIVE` requires `financeSetupStatus = COMPLETE`.

### 4.2 Value objects (key ones)

- **`OnboardingStatus`** — enum-backed VO with `canTransitionTo()`. Values: `DRAFT`, `PENDING_APPROVAL`, `PENDING_FINANCE`, `ACTIVE`, `REJECTED`, `DEACTIVATED`. (UI "New onboarded" badge = recently entered `ACTIVE`/`PENDING_FINANCE`; treated as presentation styling, not a separate domain state.)
- **`FinanceSetupStatus`** — `NOT_STARTED`, `IN_PROGRESS`, `AWAITING_APPROVAL`, `COMPLETE`. Maps to UI "In progress / Awaiting approval / Complete".
- **`Lcn`** — format `LCN-NNNNNN`; validates pattern; generated centrally.
- **`CompanyCode`** — short code (e.g. `HRZ-1102`); pattern-validated.
- **Fee schedule** — *not a `clients` VO.* Lives in the **service-fees** module as its own aggregate: map of `ServiceType (AIR|HOTEL|CAR|RAIL|VISA|OTHER)` → `{ online: FeeExpression, offline: FeeExpression }`, where `FeeExpression` accepts amount (`¥1,200`), percent (`1.5%`), or house notation. `clients` only holds a `ServiceFeePort` reference (§1.4).
- **`PassportInfo` / `VisaInfo`** — all-optional in current scope (Step 1 fields are not required to save a draft); cross-field rule: expiry > publication when both present.
- **`CompanyProfile`** — bilingual name/address (EN + Kanji), contact, email, phone, country, tax id, industry, client category, `vip`, `handleWithCare`, internal reference.
- **`TravelPreferences` / `ServicePolicy`** — free-form preference fields from Step 3 (wireframe-level; stored as structured strings).
- **`BankingDetails`** — bank name, account holder, account no, swift, payment method (`INVOICE|CARD|...`), billing cycle (`MONTHLY|...`), credit limit, invoice email, billing address, notes.

All VOs: private constructor + static `create(props): Result<VO>` performing validation; equality by value; immutable.

### 4.3 Domain services (interfaces in domain, impl in infrastructure)

- `CompanyNameUniquenessService.isUnique(normalizedName, exceptClientId?): Promise<boolean>` — needs a repository query, so it is a domain-service **interface**, implemented in infrastructure over the ORM.
- `LcnGeneratorService.next(): Promise<Lcn>` — sequence/strategy lives in infrastructure.

### 4.4 Domain events (defined now, broker deferred)

Events implement `DomainEvent { occurredAt, aggregateId, eventName, payload }`. Recorded on the aggregate (`AggregateRoot.addEvent`), pulled and dispatched by the use case **after** successful persistence via `DomainEventDispatcher`. Default dispatcher is `InMemoryEventDispatcher` (logs + invokes in-process handlers). A `client-event.producer.ts` stub exists where the RabbitMQ adapter will later subscribe — no broker dependency added now.

Events: `ClientDraftCreated`, `ClientSubmittedForApproval`, `ClientApproved`, `ClientRejected`, `ClientFinanceStatusChanged`, `ClientDeactivated`. Payloads carry ids + the data downstream (Policy/Audit/Notification) will need.

---

## 5. Status state machine

```
                 createDraft
                     │
                     ▼
   ┌──────────────►DRAFT───────────────┐ deactivate
   │                 │ submitForApproval│
   │ (revise)        ▼                  ▼
   │          PENDING_APPROVAL ──────► DEACTIVATED
   │            │            │             ▲
   │     approve│            │reject        │ deactivate
   │            ▼            ▼              │
   │      PENDING_FINANCE   REJECTED───────►┘
   │            │            │
   │            │            └──(resubmit → DRAFT, clears rejectionReason)
   │  financeSetupStatus=COMPLETE
   │            ▼
   └────────  ACTIVE ─────────────────► DEACTIVATED
```

Transition table (enforced by `OnboardingStatus.canTransitionTo`):

| From | Allowed to |
|---|---|
| `DRAFT` | `PENDING_APPROVAL`, `DEACTIVATED` |
| `PENDING_APPROVAL` | `PENDING_FINANCE`, `REJECTED`, `DEACTIVATED` |
| `REJECTED` | `DRAFT` (resubmit/revise), `DEACTIVATED` |
| `PENDING_FINANCE` | `ACTIVE` (only when finance `COMPLETE`), `DEACTIVATED` |
| `ACTIVE` | `DEACTIVATED` |
| `DEACTIVATED` | — (terminal) |

`FinanceSetupStatus`: `NOT_STARTED → IN_PROGRESS → AWAITING_APPROVAL → COMPLETE` (forward, plus `AWAITING_APPROVAL → IN_PROGRESS` rework). Driven by `update-finance-setup-status.use-case` (Finance module / admin action). Reaching `COMPLETE` while `PENDING_FINANCE` enables (does not auto-fire) the `ACTIVE` transition; an explicit approve/activation step performs it and emits the event.

---

## 6. Application layer (use cases)

Every use case implements `UseCase<Input, Output>` with a single `execute()`; wrapped in a transaction via a `@Transactional`-style unit-of-work helper or explicit `DataSource.transaction`. Pattern per command use case:

1. Load aggregate via repository (or build via factory).
2. Invoke domain method(s) (business rules live in the aggregate/VOs — use case has no rules).
3. Persist via repository.
4. Pull recorded domain events; hand to `DomainEventDispatcher` post-commit.
5. Map to response DTO via app-mapper.

Use-case inventory:

| Use case | Trigger (UI) | Notes |
|---|---|---|
| `CreateClientDraftUseCase` | "Add New Client" → Step 1 first save | runs uniqueness + LCN gen; returns `clientId` |
| `UpdateClientSectionUseCase` | each "Next: …" / autosave (Steps 1–4) | param: section enum + payload; persists partial draft. **Step 5 (service-fees) is not handled here** — it routes through `ServiceFeePort` to the service-fees module |
| `SubmitClientForApprovalUseCase` | Step 6 "Submit for approval" | requires `isReadyForSubmission()` **and** `ServiceFeePort.hasSchedule(clientId)` = true; shared transaction with the service-fees write per §1.4 |
| `ApproveClientUseCase` | In-Approval tab action | → `PENDING_FINANCE` |
| `RejectClientUseCase` | In-Approval tab action | reason required → `REJECTED` |
| `DeactivateClientUseCase` | registry action | soft delete |
| `UpdateFinanceSetupStatusUseCase` | Finance workflow | drives finance sub-state |
| `ListClientsUseCase` | Registry/In-Approval/Rejected tabs + search | filter by status group, paginated |
| `GetClientDetailUseCase` | "View Profile" / Review step | full read model |
| `GetRegistryCountersUseCase` | dashboard cards | counts: tracked, pending approvals, rejected |
| `ParseClientTemplateUseCase` | "Upload template" | uses `TemplateParserPort`; returns prefill DTO, **no persistence** |

"Step 6 Save" (vs Submit) = keep `DRAFT` (persist only). "Submit for approval" = `submitForApproval()` transition.

---

## 7. Presentation layer & API surface

Global: `ValidationPipe` (whitelist + transform), versioned prefix `/api/v1`, response-envelope interceptor `{ data, meta }`, `DomainExceptionFilter` mapping domain exceptions → HTTP (`DuplicateCompanyName` → 409, `InvalidStatusTransition` → 409, `ClientNotFound` → 404, `ClientValidation` → 422). Guards (`AuthGuard`, scope guard mirroring legacy `IsLoginUser`) scaffolded but permissive until Role module exists.

| Method | Path | Use case | UI |
|---|---|---|---|
| `POST` | `/clients/drafts` | CreateClientDraft | Wizard start (Step 1 save) |
| `PATCH` | `/clients/:id/basic-details` | UpdateClientSection | Step 1 |
| `PATCH` | `/clients/:id/company-details` | UpdateClientSection | Step 2 |
| `PATCH` | `/clients/:id/conditions-of-use` | UpdateClientSection | Step 3 |
| `PATCH` | `/clients/:id/bank-billing` | UpdateClientSection | Step 4 |
| `PATCH` | `/clients/:id/service-fees` | → `ServiceFeePort` (service-fees module) | Step 5 — controller proxies; no `clients` persistence |
| `GET` | `/clients/:id` | GetClientDetail | Step 6 review / View Profile |
| `POST` | `/clients/:id/submit` | SubmitClientForApproval | Step 6 Submit |
| `POST` | `/clients/:id/approve` | ApproveClient | In Approval tab |
| `POST` | `/clients/:id/reject` | RejectClient | In Approval tab |
| `POST` | `/clients/:id/deactivate` | DeactivateClient | Registry action |
| `PATCH` | `/clients/:id/finance-setup` | UpdateFinanceSetupStatus | Finance workflow |
| `GET` | `/clients` | ListClients | Registry/In-Approval/Rejected (`?tab=&q=&page=&size=`) |
| `GET` | `/clients/counters` | GetRegistryCounters | Dashboard cards |
| `POST` | `/clients/template/parse` | ParseClientTemplate | Upload template |

`?tab` → status group: `registry` = {DRAFT?,PENDING_FINANCE,ACTIVE} (active, non-rejected, non-deactivated), `in-approval` = {PENDING_APPROVAL}, `rejected` = {REJECTED}. Deactivated excluded from all by default. `q` searches client/legal name + LCN + company code.

---

## 8. Infrastructure (TypeORM + PostgreSQL)

### 8.1 Table `clients` (single table now; child tables added with profile editing later)

| Column | Type | Notes |
|---|---|---|
| `id` | uuid PK | |
| `lcn` | varchar unique | `LCN-NNNNNN` |
| `company_code` | varchar | indexed |
| `company_name_en` | varchar | |
| `company_name_en_normalized` | varchar | **partial unique index** WHERE `is_active` (duplicate-name guard) |
| `company_name_kanji` | varchar null | |
| `basic_details` | jsonb | Step 1 (name/title/gender/dob/profession/passport/visa) |
| `company_profile` | jsonb | Step 2 remaining fields + flags |
| `travel_preferences` | jsonb | Step 3 |
| `service_policy` | jsonb | Step 3 |
| `banking_details` | jsonb | Step 4 |
| _(no fee column)_ | — | Step 5 data lives in the **service-fees** module's tables (§1.4) |
| `onboarding_status` | varchar | indexed |
| `finance_setup_status` | varchar | indexed |
| `rejection_reason` | text null | |
| `is_active` | boolean default true | soft-delete flag |
| `created_at/by`, `updated_at/by`, `deactivated_at` | timestamptz / varchar | audit |

Rationale for `jsonb` per step now: the wizard sections are wide and still "wireframe-level"; jsonb keeps migrations cheap during onboarding-only phase. When profile editing arrives, normalized columns/child tables (`client_passport`, `client_visa`, `client_card`, …) replace the relevant jsonb blobs — the ORM mapper isolates this so the **domain & application layers do not change**.

Indexes: `lcn` unique; `(onboarding_status)`; `(finance_setup_status)`; partial unique on `company_name_en_normalized WHERE is_active`; trigram/`ILIKE` support index for search on name + lcn + company_code.

### 8.2 Repository

`ClientRepository` (domain interface): `save(client)`, `findById(id)`, `findByCompanyNameNormalized(name)`, `list(filter): Page<Client>`, `counts(): {tracked, pendingApproval, rejected}`. **No `delete`.** Impl uses `client.orm-mapper.ts` for bidirectional aggregate ↔ row mapping; the aggregate never leaks ORM types.

### 8.3 Other infra

- `CompanyNameUniquenessServiceImpl` → repository query on normalized name.
- `LcnGeneratorServiceImpl` → Postgres sequence (`client_lcn_seq`) formatted to `LCN-%06d`.
- `XlsxTemplateParser` implements `TemplateParserPort` (lib choice: `exceljs`/`papaparse`; added in package.json during impl).
- `ClientEventProducer` (stub): receives dispatched events, logs structured payload; `// TODO: bind RabbitMQ exchange` marker. No `amqplib`/`@nestjs/microservices` dependency added in this phase.

---

## 9. Cross-cutting

- **Config**: `@nestjs/config` + Joi schema; `database.config.ts` builds `TypeOrmModuleOptions` from env; migrations run via `typeorm` CLI script (not `synchronize`).
- **Transactions**: each command use case wraps repository writes + event recording in one DB transaction; events dispatched only after commit (no event on rollback). This fixes the legacy "double-write history not transactional" risk.
- **Logging**: Nest `Logger` + request-id interceptor; structured proc-style log on validation failure (mirrors legacy `NLogService.PrintProcLog` intent — list missing required fields).
- **Error model**: domain throws typed exceptions; filter maps to RFC-7807-ish body `{ code, message, details }`.
- **Security notes carried from legacy**: server-side authz must not trust client-supplied scope params (legacy "direct-link tampering" risk) — enforced in use cases/guards, not just presentation; all output DTO strings safe by construction (JSON API, no server-rendered HTML — removes legacy stored-XSS picker risk).

---

## 10. NestJS module wiring

`ClientManagementModule`:
- `imports`: `TypeOrmModule.forFeature([ClientOrmEntity])`, shared `DomainEventsModule`.
- DI tokens (string/symbol) bind domain interfaces → infra impls:
  `CLIENT_REPOSITORY → ClientRepositoryImpl`, `COMPANY_NAME_UNIQUENESS → ...Impl`, `LCN_GENERATOR → ...Impl`, `TEMPLATE_PARSER → XlsxTemplateParser`, `DOMAIN_EVENT_DISPATCHER → InMemoryEventDispatcher`, `SERVICE_FEE_PORT → ServiceFeeModuleAdapter` (in-process adapter calling the `service-fees` module use case; swappable for an outbox/remote adapter on extraction).
- `imports` also includes `ServiceFeesModule` (exports only its application use case + a narrow facade consumed by the adapter — `clients` never imports `service-fees` domain).
- `providers`: all use cases + bindings; `controllers`: the four controllers.
- Registered in `app.module.ts` alongside `ConfigModule`, `TypeOrmModule.forRootAsync`.

---

## 11. Validation & business-rule placement

| Rule | Where |
|---|---|
| DTO shape, types, formats | presentation DTO (`class-validator`) |
| VO format (LCN pattern, fee expression, date sanity) | domain VO `create()` |
| Cross-field (passport expiry > issue; ACTIVE needs finance COMPLETE) | aggregate |
| Duplicate company name | aggregate via `CompanyNameUniquenessService` |
| Status transition legality | `OnboardingStatus` VO + aggregate |
| Submission completeness (required set per step) | aggregate `isReadyForSubmission()` delegating to each VO's `isComplete()` |
| Finance/AR validation of fee values "in production" | out of scope (UI explicitly defers to Finance) |

No business logic in controllers or use cases — only orchestration. This is the explicit fix for the legacy `RecClient` god-object: rules are distributed across small VOs + the aggregate.

---

## 12. Testing strategy

- **Domain unit tests** (no Nest, no DB): aggregate transitions, every VO validation, state-machine table, invariants (duplicate name via fake service, illegal transition, reject-without-reason).
- **Application tests**: use cases with in-memory repository fakes; assert events recorded & dispatched post-commit; assert no event on failure.
- **Infrastructure tests**: repository impl against Postgres (Testcontainers); partial-unique-index behavior; mapper round-trip.
- **E2E**: wizard happy path (draft → 5 sections → submit → approve → finance complete → active), reject path, duplicate-name 409, deactivate then absent from list, counters correctness, template parse prefill.
- Coverage gates kept on domain + application (highest value layers).

---

## 13. Extensibility (how the deferred scope plugs in later)

Designed so additions are **additive, not restructuring**:

1. **Profile editing (passport/visa/cards + history)**: introduce child entities under `Client` aggregate (`domain/entities/*`), new VOs, replace corresponding `jsonb` columns with child tables + mappers. New use cases `AddClientVisa`, `RemoveClientCard`, … New `*_history` append tables behind the same transactional use-case pattern (fixes legacy non-transactional double-write). Domain/application interfaces untouched.
2. **Relationships (代行 agent / 同行 companion / arranger)**: separate aggregates (`ClientAgent`, `ClientCompanion`) referencing `ClientId`, own repositories, own use cases & controller routes — no change to `Client`.
3. **Approval-route engine**: a `ClientApprovalRoute` aggregate + policy domain service consumed by `SubmitClientForApproval` via a port; default impl = single-step until route engine exists.
4. **Authority / `RecAuthrity`**: a `RoleUserManagement` module exposing a guard + policy service; presentation guards already scaffolded to consume it.
5. **Messaging broker**: implement the RabbitMQ adapter behind existing `DomainEventDispatcher`/`ClientEventProducer` interfaces; bind in module — zero domain/application change. Downstream Policy/Audit/Notification consumers subscribe to already-defined event names.
6. **Bulk profile upload**: extend `ParseClientTemplate` + a `BulkOnboardClients` use case reusing `CreateClientDraft` per row inside a batch transaction.

---

## 14. Phased delivery / task breakdown

**Phase 0 – Foundation**
1. Add deps: `@nestjs/config`, `@nestjs/typeorm` + `typeorm` + `pg`, `class-validator`/`class-transformer`, `joi`, parser lib.
2. `config/` (app, database), datasource + migration scripts, `app.module` wiring.
3. `shared/` base classes (aggregate-root, VO, entity, Result, domain-event, dispatcher), ESLint boundary rules.

**Phase 1 – Domain core**
4. VOs (status machine first), `Client` aggregate + factory + lifecycle + invariants, domain services & repository interfaces, events, exceptions. Full domain unit tests.

**Phase 2 – Persistence**
5. ORM entity + migration, mapper, repository impl, uniqueness + LCN impls. Testcontainers tests.

**Phase 3 – Application**
6. All use cases + app mapper/assembler + ports + orchestrator. Application tests with fakes.
6a. `ServiceFeePort` interface + a **fake adapter** so `clients` Phase 3–5 proceed independently of the real `service-fees` module. Replace with the real `ServiceFeeModuleAdapter` once `service-fees` lands (parallel track); Submit's shared-transaction path validated then.

**Phase 4 – Presentation**
7. DTOs + 4 controllers + validation pipe + exception filter + response interceptor + permissive guards. Wire `ClientManagementModule`.

**Phase 5 – Wizard support & registry**
8. Template-parse endpoint, list filters/search/pagination, counters. E2E suite.

**Phase 6 – Hardening**
9. Logging/request-id, error envelope, transaction boundaries verified, event-dispatch-post-commit verified, security review of scope checks. Documentation update.

---

## 15. Assumptions & open questions

Assumptions made (flag if wrong):

- "New onboarded" badge is a presentation styling of a recently-entered active/pending-finance client, **not** a distinct domain state.
- Service-fee values are stored as validated-but-flexible expressions; authoritative Finance validation is out of scope (per UI note).
- LCN is system-generated at draft creation (UI shows LCN before activation in the registry).
- Soft-deactivated clients are excluded from all three tabs and from the duplicate-name uniqueness check (name freed on deactivation).
- Saving an incomplete draft is allowed; only **Submit** enforces the required-field set.

Decided (recorded, not open):

- **3-module split** — `clients` (this plan), `service-fees`, `client-reports` as sibling bounded contexts (§1.4).
- **Option A** — `service-fees` is the single source of truth for all fee data including the onboarding (Step 5) schedule; `clients` holds no fee data and delegates via `ServiceFeePort`. Onboarding consistency via shared local transaction now, outbox/saga when `service-fees` is extracted.

Open questions for product/Finance before/within Phase 1:

1. Exact required-field set per step for `isReadyForSubmission()` (legacy used customer-configurable `BTM_getCustomerControlWebProfile`; is per-customer required-field config in scope now or a constant set?).
2. Is `companyCode` user-entered or derived/generated? (UI shows it but the wizard has no explicit field.)
3. Who/what advances `FinanceSetupStatus` — a Finance module action, or a field in this wizard?
4. Does "Approve" go straight to `PENDING_FINANCE`, or is there an interim state before finance starts?
5. Multi-tenancy scoping key for the uniqueness rule — is company-name uniqueness global or per owning customer/tenant (`bcd-japan-tenant-policy-service` implies tenant scoping)?

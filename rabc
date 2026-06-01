# Access Management & Authorization Guide — Tenant Policy Service

This document describes the access-management data model for **`bcd-japan-tenant-policy-service`** and gives a complete, step-by-step guide for building the authorization guard, given that **authentication (token verification) is done at the Kong API Gateway** and the service only receives the resolved `keycloakId`.

> Scope: this guide is specific to the tenant-policy-service. It assumes Kong has already validated the JWT and rejected unauthenticated calls before they reach this service.

---

## 1. Data Model

All access-management tables live in the `tenant_policy` schema (see `DB_SCHEMA` in `app.module.ts`). There are five tables.

### 1.1 Tables

#### `users` — `UserOrmEntity`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | Generated |
| `email` | varchar(255) | Unique (`idx_users_email`) |
| `firstName` | varchar(100) | |
| `lastName` | varchar(100) | |
| `status` | enum `UserStatus` | `ACTIVE` \| `INACTIVE` \| `SUSPENDED`, default `ACTIVE` (`idx_users_status`) |
| `externalProfileId` | varchar, nullable | **This holds the Keycloak ID.** This is the link between Kong's token subject and a local user. |
| `createdAt` / `updatedAt` | timestamp | Managed |
| `deletedAt` | timestamp, nullable | Soft delete |

#### `roles` — `RoleOrmEntity`
| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `name` | varchar(150) | Unique (`idx_roles_name`) |
| `description` | text, nullable | |
| `type` | enum `RoleType` | `SYSTEM` \| `CUSTOM` |
| `status` | enum `RoleStatus` | `ACTIVE` \| `INACTIVE`, default `ACTIVE` (`idx_roles_status`) |
| timestamps + `deletedAt` | | Soft delete |

#### `modules` — `ModuleOrmEntity`
A "module" is a protected functional area of the product (e.g. `CLIENT_MANAGEMENT`, `POLICY`).
| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `code` | varchar(100) | Unique (`idx_modules_code`) — stable identifier used in code |
| `name` | varchar(150) | Display name |
| `description` | text, nullable | |
| `displayOrder` | int, default 0 | (`idx_modules_display_order`) |
| `isActive` | boolean, default true | |
| timestamps | | (no soft delete) |

#### `user_roles` — `UserRoleOrmEntity` (join: user ↔ role)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `userId` | uuid (FK → users) | `onDelete: CASCADE` |
| `roleId` | uuid (FK → roles) | `onDelete: CASCADE` |
| `assignedBy` | uuid, nullable | Who granted the role |
| `assignedAt` | timestamp | |
| | | Unique pair `(userId, roleId)` (`uq_user_role`); `idx_user_role_user` |

#### `role_module_permissions` — `RoleModulePermissionOrmEntity` (join: role ↔ module + access level)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid (PK) | |
| `roleId` | uuid (FK → roles) | `onDelete: CASCADE` |
| `moduleId` | uuid (FK → modules) | `onDelete: CASCADE` |
| `accessLevel` | enum `AccessLevel` | `NONE` \| `READ_ONLY` \| `READ_WRITE` |
| | | Unique pair `(roleId, moduleId)` (`uq_role_module_permission`); `idx_role_module_permission_role` |

### 1.2 Relations (ER diagram)

```mermaid
erDiagram
    USERS ||--o{ USER_ROLES : assigned
    ROLES ||--o{ USER_ROLES : grants
    ROLES ||--o{ ROLE_MODULE_PERMISSIONS : defines
    MODULES ||--o{ ROLE_MODULE_PERMISSIONS : secured_by

    USERS {
        uuid id PK
        varchar email UK
        varchar first_name
        varchar last_name
        varchar status
        varchar external_profile_id "Keycloak ID"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }

    ROLES {
        uuid id PK
        varchar name UK
        text description
        varchar type "SYSTEM | CUSTOM"
        varchar status "ACTIVE | INACTIVE"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }

    MODULES {
        uuid id PK
        varchar code UK
        varchar name
        text description
        int display_order
        boolean is_active
        timestamptz created_at
        timestamptz updated_at
    }

    USER_ROLES {
        uuid id PK
        uuid user_id FK
        uuid role_id FK
        uuid assigned_by
        timestamptz assigned_at
    }

    ROLE_MODULE_PERMISSIONS {
        uuid id PK
        uuid role_id FK
        uuid module_id FK
        varchar access_level "NONE | READ_ONLY | READ_WRITE"
    }
```

> **Note vs. a `department_id` / direct `role_id` design:** this service has **no `departments` table**, and a user is *not* linked to a single role by an FK column. Instead, users and roles form a **many-to-many** relationship through the `user_roles` join table (a user can hold several roles), and each role's access is defined per-module through `role_module_permissions`. The join tables carry no `created_at`/`updated_at` in the current entities (`user_roles` has `assigned_at`/`assigned_by`; `role_module_permissions` has neither).

ASCII version of the same relations:

```
                      ┌──────────────────────┐
                      │        users         │
                      │──────────────────────│
                      │ id (PK)              │
                      │ email (unique)       │
                      │ externalProfileId ◄──┼──── Keycloak ID lives here
                      │ status               │
                      └──────────┬───────────┘
                                 │ 1
                                 │
                                 │ N
                      ┌──────────┴───────────┐
                      │      user_roles      │   (M:N bridge: user ↔ role)
                      │──────────────────────│
                      │ id (PK)              │
                      │ userId (FK)          │
                      │ roleId (FK)          │
                      │ assignedBy           │
                      │ assignedAt           │
                      └──────────┬───────────┘
                                 │ N
                                 │
                                 │ 1
                      ┌──────────┴───────────┐
                      │        roles         │
                      │──────────────────────│
                      │ id (PK)              │
                      │ name (unique)        │
                      │ type / status        │
                      └──────────┬───────────┘
                                 │ 1
                                 │
                                 │ N
                ┌────────────────┴────────────────┐
                │     role_module_permissions      │  (M:N bridge: role ↔ module
                │──────────────────────────────────│   + accessLevel payload)
                │ id (PK)                          │
                │ roleId (FK)                      │
                │ moduleId (FK)                    │
                │ accessLevel (NONE/RO/RW)         │
                └────────────────┬─────────────────┘
                                 │ N
                                 │
                                 │ 1
                      ┌──────────┴───────────┐
                      │       modules        │
                      │──────────────────────│
                      │ id (PK)              │
                      │ code (unique)        │
                      │ name / isActive      │
                      └──────────────────────┘
```

**Effective permission resolution** (already implemented in `GetUserPermissionsUseCase`):

```
user → user_roles → roles → role_module_permissions → modules
```

A user inherits the union of all `(moduleCode, accessLevel)` pairs from every role assigned to them. When two roles grant different levels on the same module, you take the **highest** (`READ_WRITE` > `READ_ONLY` > `NONE`).

---

## 2. The Authentication Boundary (Kong)

```
┌────────┐   JWT (Bearer)   ┌──────────────┐   keycloakId header   ┌────────────────────────┐
│ Client │ ───────────────► │     Kong     │ ────────────────────► │ tenant-policy-service  │
└────────┘                  │ (verifies    │                       │ (this service)         │
                            │  JWT, OIDC)  │                       │ - trusts the header    │
                            └──────────────┘                       │ - authorizes via RBAC  │
                                                                   └────────────────────────┘
```

Kong terminates authentication. By the time a request reaches this service:

- The JWT signature, expiry, issuer and audience have **already been validated**. This service does **not** re-verify the token.
- Kong forwards the authenticated principal to upstream. Depending on the Kong plugin, the Keycloak subject arrives as either:
  - an **HTTP header** Kong injects (recommended — simplest), e.g. `X-Authenticated-Sub`, `X-Userinfo` (base64 JSON), or a custom `X-Keycloak-Id`; **or**
  - the **raw JWT** still in `Authorization: Bearer …`, from which we decode (not verify) the `sub` claim.

> **Decision you must make with the platform team:** which header Kong injects, and whether the gateway is network-isolated so the header cannot be spoofed by external clients. This service *trusts* that header, so Kong must **strip any client-supplied copy** of it and only the gateway may set it. If the service is reachable without going through Kong, the whole model is bypassable.

This guide assumes Kong injects a header named `x-keycloak-id` (adjust the constant if your gateway uses a different one). If you only get the raw JWT, see §6.

---

## 3. What needs to be built

The repository already has the domain, application, infra and presentation layers for users/roles/modules/permissions. To add the guard you need to add **five new pieces**:

1. A repository lookup to find a user by `externalProfileId` (Keycloak ID). *(does not exist yet)*
2. An application use-case that resolves the authenticated user + their effective permissions.
3. A request-scoped "current principal" type + a way to attach it to the request.
4. An `AuthGuard` that reads the header, resolves the user, and populates the request.
5. A `PermissionsGuard` + `@RequirePermission()` decorator for per-route module/access-level checks, plus a `@CurrentUser()` param decorator.

Below is the full implementation. Paths are relative to `src/`.

---

## 4. Step-by-step implementation

### Step 1 — Add a lookup by Keycloak ID to the user repository

**`modules/access-management/application/interfaces/user.repository.ts`** — add one method:

```ts
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findByExternalProfileId(externalProfileId: string): Promise<User | null>; // NEW
  list(dto: ListUsersDto): Promise<{ data: User[]; total: number }>;
  save(user: User): Promise<User>;
}
```

**`modules/access-management/infrastructure/repositories/user.repository.impl.ts`** — implement it (mirror `findByEmail`):

```ts
async findByExternalProfileId(externalProfileId: string): Promise<User | null> {
  const entity = await this.repository.findOne({
    where: { externalProfileId },
    relations: { roles: true },
  });

  return entity ? UserOrmMapper.toDomain(entity) : null;
}
```

> Recommended: add a unique index on `externalProfileId` so the lookup is fast and one Keycloak identity maps to exactly one local user. On `UserOrmEntity`:
> ```ts
> @Index('idx_users_external_profile_id', ['externalProfileId'], {
>   unique: true,
>   where: '"externalProfileId" IS NOT NULL', // partial: allow many NULLs
> })
> ```

### Step 2 — A use-case that resolves the authenticated principal

Create **`modules/access-management/application/use-cases/auth/resolve-principal.use-case.ts`**. It loads the user by Keycloak ID, rejects inactive users, and flattens their effective permissions into a `Map<moduleCode, AccessLevel>` (highest level wins).

```ts
import { Inject, Injectable } from '@nestjs/common';
import { AccessLevel } from '../../../domain/enums/access-level.enum';
import { UserStatus } from '../../../domain/enums/user-status.enum';
import {
  MODULE_REPOSITORY,
  ROLE_MODULE_PERMISSION_REPOSITORY,
  USER_REPOSITORY,
} from '../../constants/token';
import type { IModuleRepository } from '../../interfaces/module.repository';
import type { IRoleModulePermissionRepository } from '../../interfaces/role-module-permission.repository';
import type { IUserRepository } from '../../interfaces/user.repository';

export interface AuthenticatedPrincipal {
  userId: string;
  email: string;
  externalProfileId: string;
  roleIds: string[];
  // moduleCode -> highest granted access level
  permissions: Map<string, AccessLevel>;
}

const RANK: Record<AccessLevel, number> = {
  [AccessLevel.NONE]: 0,
  [AccessLevel.READ_ONLY]: 1,
  [AccessLevel.READ_WRITE]: 2,
};

@Injectable()
export class ResolvePrincipalUseCase {
  constructor(
    @Inject(USER_REPOSITORY) private readonly users: IUserRepository,
    @Inject(MODULE_REPOSITORY) private readonly modules: IModuleRepository,
    @Inject(ROLE_MODULE_PERMISSION_REPOSITORY)
    private readonly rolePermissions: IRoleModulePermissionRepository,
  ) {}

  async execute(externalProfileId: string): Promise<AuthenticatedPrincipal | null> {
    const user = await this.users.findByExternalProfileId(externalProfileId);
    if (!user || !user.id) return null;

    // A suspended/inactive user is authenticated by Kong but not authorized here.
    if (user.status !== UserStatus.ACTIVE) return null;

    const roleIds = user.roles.map((r) => r.roleId);
    const permissions = new Map<string, AccessLevel>();

    if (roleIds.length > 0) {
      const perms = await this.rolePermissions.findByRoleIds(roleIds);

      // Resolve module codes; cache to avoid N lookups for repeated modules.
      const moduleCache = new Map<string, string>();
      for (const p of perms) {
        let code = moduleCache.get(p.moduleId);
        if (!code) {
          const module = await this.modules.findById(p.moduleId);
          if (!module) continue;
          code = module.code;
          moduleCache.set(p.moduleId, code);
        }

        const current = permissions.get(code);
        if (!current || RANK[p.accessLevel] > RANK[current]) {
          permissions.set(code, p.accessLevel);
        }
      }
    }

    return {
      userId: user.id,
      email: user.email,
      externalProfileId,
      roleIds,
      permissions,
    };
  }
}
```

> Performance note: this hits the DB on every request. Once it works, wrap `execute` in a short-lived cache (e.g. `@nestjs/cache-manager`, 30–60s TTL keyed by `externalProfileId`), and invalidate on role/permission changes. Start correct, then optimize.

### Step 3 — Principal type + header constant

Create **`common/auth/auth.constants.ts`**:

```ts
/** Header Kong injects after verifying the JWT. Keep in sync with the gateway config. */
export const KEYCLOAK_ID_HEADER = 'x-keycloak-id';

/** Metadata key for the @RequirePermission() decorator. */
export const PERMISSION_KEY = 'required_permission';
```

Augment Express's `Request` so `req.principal` is typed. Create **`common/auth/principal.types.ts`**:

```ts
import type { AuthenticatedPrincipal } from '../../modules/access-management/application/use-cases/auth/resolve-principal.use-case';

declare global {
  // eslint-disable-next-line @typescript-eslint/no-namespace
  namespace Express {
    interface Request {
      principal?: AuthenticatedPrincipal;
    }
  }
}

export {};
```

### Step 4 — The `AuthGuard`

Reads the trusted header, resolves the principal, attaches it to the request. Rejects when the header is missing or the user is unknown/inactive.

Create **`common/auth/auth.guard.ts`**:

```ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import type { Request } from 'express';
import { KEYCLOAK_ID_HEADER } from './auth.constants';
import { ResolvePrincipalUseCase } from '../../modules/access-management/application/use-cases/auth/resolve-principal.use-case';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly resolvePrincipal: ResolvePrincipalUseCase) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest<Request>();

    const headerValue = req.headers[KEYCLOAK_ID_HEADER];
    const keycloakId = Array.isArray(headerValue) ? headerValue[0] : headerValue;

    if (!keycloakId) {
      // Should never happen if Kong is in front; means request bypassed the gateway.
      throw new UnauthorizedException('Missing authenticated principal');
    }

    const principal = await this.resolvePrincipal.execute(keycloakId);
    if (!principal) {
      // Authenticated at Kong, but no active local user is mapped to this identity.
      throw new UnauthorizedException('User is not provisioned or is inactive');
    }

    req.principal = principal;
    return true;
  }
}
```

### Step 5 — `@RequirePermission()` decorator + `PermissionsGuard`

The decorator tags a route with `(moduleCode, minimum accessLevel)`. The guard checks the resolved principal against it.

Create **`common/auth/require-permission.decorator.ts`**:

```ts
import { SetMetadata } from '@nestjs/common';
import { AccessLevel } from '../../modules/access-management/domain/enums/access-level.enum';
import { PERMISSION_KEY } from './auth.constants';

export interface RequiredPermission {
  moduleCode: string;
  level: AccessLevel; // minimum level required
}

export const RequirePermission = (moduleCode: string, level: AccessLevel) =>
  SetMetadata(PERMISSION_KEY, { moduleCode, level } as RequiredPermission);
```

Create **`common/auth/permissions.guard.ts`**:

```ts
import {
  CanActivate,
  ExecutionContext,
  ForbiddenException,
  Injectable,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import type { Request } from 'express';
import { AccessLevel } from '../../modules/access-management/domain/enums/access-level.enum';
import { PERMISSION_KEY, type RequiredPermission } from './auth.constants';
import type { RequiredPermission as Req } from './require-permission.decorator';

const RANK: Record<AccessLevel, number> = {
  [AccessLevel.NONE]: 0,
  [AccessLevel.READ_ONLY]: 1,
  [AccessLevel.READ_WRITE]: 2,
};

@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<Req>(PERMISSION_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    // No @RequirePermission on the route → authentication is enough.
    if (!required) return true;

    const req = context.switchToHttp().getRequest<Request>();
    const principal = req.principal;

    // AuthGuard must run first; this is a safety net.
    if (!principal) throw new ForbiddenException('Not authenticated');

    const granted = principal.permissions.get(required.moduleCode) ?? AccessLevel.NONE;

    if (RANK[granted] < RANK[required.level]) {
      throw new ForbiddenException(
        `Requires ${required.level} on ${required.moduleCode}`,
      );
    }

    return true;
  }
}
```

> Note: `PERMISSION_KEY` is exported from `auth.constants.ts`; import the `RequiredPermission` type from the decorator file as shown to keep a single source of truth.

### Step 6 — `@CurrentUser()` param decorator (convenience)

So controllers can read the principal without touching `req`. Create **`common/auth/current-user.decorator.ts`**:

```ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import type { Request } from 'express';
import type { AuthenticatedPrincipal } from '../../modules/access-management/application/use-cases/auth/resolve-principal.use-case';

export const CurrentUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): AuthenticatedPrincipal | undefined => {
    return ctx.switchToHttp().getRequest<Request>().principal;
  },
);
```

### Step 7 — Wire it up

**Register the use-case and guards** in `AccessManagementModule` (`modules/access-management/access-management.module.ts`). Add `ResolvePrincipalUseCase` and `AuthGuard` to `providers`, and export `ResolvePrincipalUseCase` + `AuthGuard` so other feature modules can use them:

```ts
providers: [
  // ...existing...
  ResolvePrincipalUseCase,
  AuthGuard,
],
exports: [
  // ...existing repository tokens...
  ResolvePrincipalUseCase,
  AuthGuard,
],
```

**Apply `AuthGuard` globally** so every route requires a resolved principal, and apply `PermissionsGuard` globally too (it is a no-op on routes without `@RequirePermission`). In `app.module.ts`:

```ts
import { APP_GUARD } from '@nestjs/core';
import { AuthGuard } from './common/auth/auth.guard';
import { PermissionsGuard } from './common/auth/permissions.guard';

@Module({
  imports: [ /* ...existing... AccessManagementModule must be imported... */ ],
  controllers: [HealthController],
  providers: [
    { provide: APP_GUARD, useClass: AuthGuard },        // runs first
    { provide: APP_GUARD, useClass: PermissionsGuard }, // runs second
  ],
})
export class AppModule {}
```

> Global guards run in the order they are listed in `providers`, so `AuthGuard` (populates `req.principal`) must come before `PermissionsGuard` (reads it).
>
> `AuthGuard` depends on `ResolvePrincipalUseCase`, which lives in `AccessManagementModule`. Since it is exported there and `AccessManagementModule` is imported in `AppModule`, the global `APP_GUARD` instance can inject it. `PermissionsGuard` only needs `Reflector` (always available).

**Allow public routes** (e.g. health check). Add a `@Public()` decorator and have `AuthGuard` skip when present:

```ts
// common/auth/public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'is_public';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

Then inject `Reflector` into `AuthGuard` and short-circuit:

```ts
const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
  context.getHandler(),
  context.getClass(),
]);
if (isPublic) return true;
```

Mark `HealthController` (and Swagger if open) with `@Public()`.

### Step 8 — Use it in controllers

```ts
import { RequirePermission } from 'src/common/auth/require-permission.decorator';
import { CurrentUser } from 'src/common/auth/current-user.decorator';
import { AccessLevel } from '../../domain/enums/access-level.enum';
import type { AuthenticatedPrincipal } from '../../application/use-cases/auth/resolve-principal.use-case';

@Controller('users')
export class UsersController {
  // Reading the user list needs READ_ONLY on ACCESS_MANAGEMENT
  @Get()
  @RequirePermission('ACCESS_MANAGEMENT', AccessLevel.READ_ONLY)
  findAll(@Query() query: ListUsersQueryDto) { /* ... */ }

  // Mutations need READ_WRITE
  @Post()
  @RequirePermission('ACCESS_MANAGEMENT', AccessLevel.READ_WRITE)
  create(
    @Body() dto: CreateUserRequestDto,
    @CurrentUser() actor: AuthenticatedPrincipal, // who is performing the action
  ) {
    // e.g. record actor.userId as assignedBy
  }
}
```

The `'ACCESS_MANAGEMENT'` string must match a `modules.code` row in the DB. Seed your modules table with the codes you reference in decorators.

---

## 5. Request flow (end to end)

```
1. Client → Kong with `Authorization: Bearer <jwt>`
2. Kong verifies JWT (sig/exp/iss/aud). On failure → 401 (never reaches us).
3. Kong strips any client `x-keycloak-id`, injects the real `sub` as `x-keycloak-id`.
4. Request hits this service.
5. AuthGuard:
     - @Public()? → allow.
     - read `x-keycloak-id` → ResolvePrincipalUseCase
        → users.findByExternalProfileId
        → reject if missing / not ACTIVE  (401)
        → build permissions Map<moduleCode, AccessLevel>
     - attach req.principal.
6. PermissionsGuard:
     - route has @RequirePermission(module, level)?
        - no  → allow
        - yes → compare principal.permissions[module] >= level
                 → below → 403, else allow.
7. Controller runs; @CurrentUser() gives the principal.
```

---

## 6. Variant: Kong forwards the raw JWT instead of a header

If the gateway leaves `Authorization: Bearer <jwt>` in place and does not inject a clean id header, decode (do **not** verify — Kong already did) the `sub` claim inside `AuthGuard`:

```ts
import { Buffer } from 'node:buffer';

function extractSub(authHeader?: string): string | null {
  if (!authHeader?.startsWith('Bearer ')) return null;
  const token = authHeader.slice('Bearer '.length);
  const [, payload] = token.split('.');
  if (!payload) return null;
  try {
    const json = JSON.parse(Buffer.from(payload, 'base64url').toString('utf8'));
    return typeof json.sub === 'string' ? json.sub : null;
  } catch {
    return null;
  }
}
```

Use the returned `sub` as the `externalProfileId`. **Only do this when the service sits strictly behind Kong**, because we are not validating the signature here — we are trusting the gateway. If the service could be reached directly, you must verify the JWT signature against Keycloak's JWKS instead (add `jose`/`jwks-rsa`), which re-introduces real authentication into this service.

---

## 7. Provisioning note (linking Keycloak ↔ local user)

The guard fails for any Keycloak identity that has no matching `users.externalProfileId`. You need a provisioning path so a first-time-authenticated user gets a local row:

- **Pre-provisioned (recommended for B2B/admin):** an admin creates the user (`POST /users`) and sets `externalProfileId` to the Keycloak ID via `User.linkExternalProfile()`. Until then the user is authenticated by Kong but gets `401 "not provisioned"` here. (`create-user.request.dto` would need an `externalProfileId`/`keycloakId` field, or a dedicated `PATCH /users/:id/link-external` endpoint.)
- **Just-in-time:** on first request, if no local user exists, create one from JWT claims (email/name) and link it. This requires Kong to forward those claims (header or full JWT) and a JIT branch inside `ResolvePrincipalUseCase`.

Pick one with the team; the guard code above treats "no mapped active user" as unauthorized, which is the safe default.

---

## 8. Checklist

- [ ] `findByExternalProfileId` added to `IUserRepository` + impl, with a unique partial index.
- [ ] `ResolvePrincipalUseCase` created and registered/exported in `AccessManagementModule`.
- [ ] `common/auth/`: constants, principal types, `AuthGuard`, `PermissionsGuard`, `@RequirePermission`, `@CurrentUser`, `@Public`.
- [ ] Global `APP_GUARD`s registered in `AppModule` in the right order (Auth → Permissions).
- [ ] `HealthController` (and Swagger, if open) marked `@Public()`.
- [ ] `modules` table seeded with the codes referenced in `@RequirePermission`.
- [ ] Confirmed with platform team: exact Kong header name, header stripping, and network isolation so the service is only reachable through Kong.
- [ ] Provisioning strategy chosen (pre-provisioned vs JIT).
- [ ] (Later) short-TTL cache on principal resolution + invalidation on role/permission change.

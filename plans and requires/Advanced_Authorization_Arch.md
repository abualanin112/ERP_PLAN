# Advanced Authorization Architecture

## CASL + Scoped Queries + Dynamic Permissions

### Express.js + Prisma + PostgreSQL

> This document defines an advanced modern authorization architecture for a scalable PropTech ERP/CRM platform using CASL as the primary authorization engine.

---

# Core Philosophy

The system combines:

- hardcoded core security rules
- dynamic database-driven permissions
- scoped database queries
- centralized authorization logic
- frontend-aware permissions

Authorization should remain:

- centralized
- maintainable
- scalable
- observable
- performant

Avoid:

- scattered role checks
- duplicated security logic
- frontend-only filtering
- giant middleware authorization

---

# High-Level Architecture

```txt
Frontend
   ↓
JWT Authentication
   ↓
Passport Middleware
   ↓
ALS Context
   ↓
CASL Ability Factory
   ↓
Prisma Scoped Queries
   ↓
PostgreSQL
```

Additional Infrastructure:

```txt
Pino Logging
Audit Logs
Background Workers
```

---

# Authorization Philosophy

The system uses:

- RBAC
- ABAC
- Scoped Queries
- Policy-as-Code

CASL becomes:

- the ability engine
- the policy engine
- the query scope generator
- the frontend permission engine

---

# Hybrid Authorization Strategy

## Protected Core + Dynamic Roles

The system combines:

# 1. Hardcoded Core Roles

These roles:

- never come from database permissions
- are protected from admin modification
- always exist
- define core platform security

Examples:

```txt
SuperAdmin
System
BasicTenant
```

---

# 2. Dynamic Roles

These roles:

- are configurable
- stored in PostgreSQL
- optionally cached later when authorization queries become a measurable bottleneck
- editable from admin panel

Examples:

```txt
BranchManager
SalesAgent
Marketing
Accountant
```

---

# Why Hybrid Authorization?

Because enterprise systems need:

- immutable platform security
- customizable business permissions

This architecture prevents:

- accidental privilege escalation
- admin panel misconfiguration
- critical role corruption

---

# Prisma Schema

## User Model

```prisma
model User {
  id            String @id @default(uuid())

  email         String @unique

  passwordHash  String?

  provider      String

  roleId        String

  tenantId      String

  branchId      String?

  regionId      String?

  role          Role @relation(
    fields: [roleId],
    references: [id]
  )
}
```

---

## Role Model

```prisma
model Role {
  id            String @id @default(uuid())

  name          String @unique

  permissions   Permission[]

  users         User[]
}
```

---

## Permission Model

```prisma
model Permission {
  id            String @id @default(uuid())

  roleId        String

  action        String

  subject       String

  conditions    Json?

  inverted      Boolean @default(false)

  role          Role @relation(
    fields: [roleId],
    references: [id],
    onDelete: Cascade
  )
}
```

---

# Denormalization Strategy

CASL performs best with:

- flat queries
- shallow conditions
- simple scopes

Avoid deep relational authorization.

---

# BAD Example

```txt
Property
   ↓
Branch
   ↓
Region
```

Then authorization checks:

```txt
property.branch.region.id
```

---

# GOOD Example

Add:

```txt
regionId
```

directly to:

```txt
Property
```

Then CASL rule becomes:

```js
can("read", "Property", {
  regionId: user.regionId,
});
```

This produces:

- faster Prisma queries
- simpler scopes
- better scalability

---

# Ability Factory

## Centralized Authorization Engine

All authorization rules MUST live here.

DO NOT place authorization logic inside:

- controllers
- services
- repositories

---

# ability.factory.js

```js
import { AbilityBuilder, createMongoAbility } from "@casl/ability";

import { getPermissionsForRole } from "./role.service.js";

export const buildAbilityFor = async (user) => {
  const { can, cannot, build } = new AbilityBuilder(createMongoAbility);

  // =========================================
  // Hardcoded Core Roles
  // =========================================

  if (user.role === "SuperAdmin") {
    can("manage", "all");

    return build();
  }

  if (user.role === "BasicTenant") {
    can("read", "RentalContract", {
      tenantId: user.id,
    });

    can("read", "MaintenanceTicket", {
      tenantId: user.id,
    });

    return build();
  }

  // =========================================
  // Dynamic Roles
  // =========================================

  const dynamicPermissions = await getPermissionsForRole(user.roleId);

  dynamicPermissions.forEach((perm) => {
    let conditions = perm.conditions
      ? JSON.stringify(perm.conditions).replace('"${user.id}"', `"${user.id}"`)
      : undefined;

    conditions = conditions ? JSON.parse(conditions) : undefined;

    if (perm.inverted) {
      cannot(perm.action, perm.subject, conditions);
    } else {
      can(perm.action, perm.subject, conditions);
    }
  });

  return build();
};
```

---

# Why This Is Powerful

This architecture provides:

- centralized security
- dynamic permissions
- future permission caching support
- frontend/backend consistency
- scoped query generation

Changing permissions updates:

- API authorization
- database scopes
- frontend visibility

immediately.

---

# Future Permission Cache

Authorization runs on every request.

Do not add Redis permission cache in the first implementation unless profiling shows authorization queries are a bottleneck. Start with indexed PostgreSQL role/permission tables. Add Redis permission cache later if permission checks become expensive under load.

---

# role.service.js

```js
export const getPermissionsForRole = async (roleId) => {
  const cacheKey = `role_permissions:${roleId}`;

  const cached = await redisClient.get(cacheKey);

  if (cached) {
    return JSON.parse(cached);
  }

  const permissions = await prisma.permission.findMany({
    where: {
      roleId,
    },
  });

  await redisClient.setEx(cacheKey, 86400, JSON.stringify(permissions));

  return permissions;
};
```

---

# Future Cache Invalidation

Whenever permissions change after permission caching is introduced:

```js
await redisClient.del(`role_permissions:${roleId}`);
```

---

# Scoped Queries with CASL

CASL generates query scopes automatically.

---

# property.service.js

```js
import { accessibleBy } from "@casl/prisma";

import { prisma } from "../../infrastructure/prisma.js";

import { getContext } from "../../middleware/context.middleware.js";

import { serializeProperty } from "./property.serializer.js";

export const getProperties = async () => {
  const { ability } = getContext();

  const properties = await prisma.property.findMany({
    where: accessibleBy(ability).Property,
  });

  return properties.map((prop) => serializeProperty(prop, ability));
};
```

---

# What accessibleBy Does

Example CASL Rule:

```js
can("read", "Property", {
  branchId: user.branchId,
});
```

Automatically becomes:

```js
where: {
  branchId: user.branchId;
}
```

inside Prisma.

---

# Field-Level Security

CASL controls:

- action permissions
- query scopes

Serializer controls:

- visible fields

---

# property.serializer.js

```js
export const serializeProperty = (property, ability) => {
  const result = {
    id: property.id,
    title: property.title,
    price: property.price,
  };

  if (ability.can("read", "internalCommission")) {
    result.internalCommission = property.internalCommission;
  }

  return result;
};
```

---

# AsyncLocalStorage Integration

ALS stores request context.

Recommended Context:

```js
{
  (reqId, traceId, userId, tenantId, ability);
}
```

---

# ALS Middleware

```js
als.run(
  {
    reqId,
    traceId,
    userId,
    tenantId,
    ability,
  },
  () => next(),
);
```

---

# Why ALS Matters

It allows:

- contextual logging
- request tracing
- automatic metadata propagation
- cleaner services

Without passing:

```js
req;
```

through every function.

---

# Logging Architecture

Use:

- Pino
- structured JSON logs

All logs should automatically include:

- reqId
- userId
- tenantId
- traceId

---

# Example Log

```json
{
  "msg": "Property updated",
  "reqId": "req_123",
  "userId": "user_5",
  "tenantId": "tenant_1"
}
```

---

# Frontend Integration

Frontend uses the SAME abilities.

---

# React Example

```jsx
{
  ability.can("delete", property) && <DeleteButton />;
}
```

---

# Why This Is Powerful

Changing CASL rules updates:

- backend authorization
- frontend UI
- database scopes

at the same time.

---

# Hiding Protected Roles

Protected roles SHOULD NOT appear in admin UI.

---

# admin.role.service.js

```js
export const getCustomizableRoles = async () => {
  return await prisma.role.findMany({
    where: {
      name: {
        notIn: ["SuperAdmin", "BasicTenant"],
      },
    },
  });
};
```

---

# Seed System

Core roles MUST be seeded automatically.

---

# prisma/seed.js

```js
await prisma.role.createMany({
  data: [
    {
      name: "SuperAdmin",
    },
    {
      name: "BasicTenant",
    },
  ],
});
```

---

# Recommended Request Flow

```txt
Request
   ↓
Passport JWT Middleware
   ↓
Load User
   ↓
Build CASL Ability
   ↓
Store ALS Context
   ↓
Controller
   ↓
Service
   ↓
accessibleBy(ability)
   ↓
Prisma Query
   ↓
Serializer
   ↓
JSON Response
```

---

# Security Best Practices

# DO

- centralize authorization
- use scoped queries
- cache permissions later only if metrics justify it
- use tenant isolation
- use ALS for context only
- use serializers
- keep core roles hardcoded
- denormalize for authorization performance

---

# DO NOT

- trust frontend filtering
- place authorization in controllers
- create deep relational authorization trees
- expose unrestricted queries
- store unlimited dynamic authorization logic
- use ALS for mutable business state

---

# Recommended Technology Stack

Authentication:

- Passport.js
- JWT
- Google OAuth

Authorization:

- CASL
- @casl/prisma

ORM:

- Prisma

Database:

- PostgreSQL

Future Cache:

- Redis

Logging:

- Pino

Context Propagation:

- AsyncLocalStorage

Observability:

- OpenTelemetry
- Prometheus
- Grafana

Error Tracking:

- Sentry

---

# Final Engineering Philosophy

Modern ERP authorization is NOT:

> "Is the user admin?"

Modern ERP authorization IS:

> "What actions can this user perform, on which resources, under which conditions, and within which data scope?"

And CASL becomes:

- the authorization language
- the query scope engine
- the frontend permission engine
- the centralized security layer

inside the entire application architecture.

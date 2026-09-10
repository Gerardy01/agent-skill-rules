---
name: backend-architecture
description: >
  Enforce the established backend architecture when generating, refactoring, or reviewing
  server-side TypeScript code. Activate this skill whenever the user asks to create a new
  API endpoint, add a feature, build a service, write a controller, set up a provider,
  define a model, or scaffold any backend module. Also trigger on phrases like "add an
  endpoint", "create a CRUD for...", "build the backend for...", "add a new entity",
  "wire up a new service", or "scaffold a module".
---

# Backend Architecture Skill

This skill defines the **layered architecture**, **manual dependency injection system**,
**folder conventions**, and **code patterns** used across all backend projects. Every piece
of generated backend code **MUST** follow these rules. No exceptions.

---

## Table of Contents

1. [Technology Stack](#1-technology-stack)
2. [Folder Structure](#2-folder-structure)
3. [Architecture Layers](#3-architecture-layers)
4. [Dependency Injection System](#4-dependency-injection-system)
5. [Layer-by-Layer Guide with Examples](#5-layer-by-layer-guide-with-examples)
   - 5.1 [Provider Layer](#51-provider-layer)
   - 5.2 [Interface Layer](#52-interface-layer)
   - 5.3 [Schema Layer](#53-schema-layer)
   - 5.4 [Model Layer](#54-model-layer)
   - 5.5 [Service Layer](#55-service-layer)
   - 5.6 [Orchestration Layer](#56-orchestration-layer)
   - 5.7 [Controller Layer](#57-controller-layer)
   - 5.8 [Route Layer](#58-route-layer)
6. [Wiring a New Feature End-to-End](#6-wiring-a-new-feature-end-to-end)
7. [Supporting Infrastructure](#7-supporting-infrastructure)
8. [API Response Format](#8-api-response-format)
9. [Error Handling Pattern](#9-error-handling-pattern)
10. [Migration Pattern](#10-migration-pattern)
11. [Checklist for New Features](#11-checklist-for-new-features)

---

## 1. Technology Stack

| Concern            | Technology                               |
|--------------------|------------------------------------------|
| Runtime            | Node.js with TypeScript                  |
| Framework          | Express 5                                |
| ORM                | Sequelize 6 (PostgreSQL dialect)         |
| Validation         | Zod 4                                    |
| Auth               | JWT (jsonwebtoken), bcrypt, OTP-based    |
| File Storage       | Cloudflare R2 (AWS S3-compatible SDK)    |
| Email              | Nodemailer                               |
| Path Aliases       | `@/*` → `src/*` via tsconfig paths       |
| Migrations         | sequelize-cli (JavaScript migration files)|

---

## 2. Folder Structure

```
src/
├── index.ts                    # Express app entry point
├── config/
│   ├── database.ts             # Sequelize instance (PostgreSQL)
│   ├── config.js               # sequelize-cli config (JS, for migrations)
│   └── index.d.ts              # Global type augmentations (e.g., Express Request.user)
├── constants/                  # Static data arrays and readonly value sets
│   ├── item.ts
│   ├── spell.ts
│   └── ...
├── interfaces/                 # DTOs and return types (one file per domain)
│   ├── IAccount.ts
│   ├── IAuth.ts
│   ├── IFaction.ts
│   └── ...
├── schema/                     # Zod validation schemas (one file per domain)
│   ├── accountSchema.ts
│   ├── authSchema.ts
│   ├── factionSchema.ts
│   └── ...
├── models/                     # Sequelize model definitions
│   ├── index.ts                # Barrel export for all models
│   ├── account/
│   │   └── account.model.ts
│   ├── workshopFaction/
│   │   └── workshopFaction.model.ts
│   └── ...
├── provider/                   # Infrastructure adapters (third-party wrappers)
│   ├── index.ts                # ★ DI Composition Root — instantiates all providers
│   ├── hashProvider.ts
│   ├── jwtProvider.ts
│   ├── storageProvider.ts
│   ├── emailProvider.ts
│   ├── cryptProvider.ts
│   ├── validatorProvider.ts
│   ├── eventPublisherProvider.ts
│   └── googleOauthProvider.ts
├── services/                   # Business logic (single-entity operations)
│   ├── index.ts                # ★ DI Composition Root — instantiates all services
│   ├── accountService.ts
│   ├── authService.ts
│   ├── fileService.ts
│   └── ...
├── orchestration/              # Cross-service coordination and transactions
│   ├── index.ts                # ★ DI Composition Root — instantiates all orchestrations
│   ├── authOrchestration.ts
│   ├── accountOrchestration.ts
│   ├── workshopFactionOrchestration.ts
│   └── ...
├── controller/                 # HTTP handlers (req/res boundary)
│   ├── authController.ts
│   ├── accountController.ts
│   ├── workshopFactionController.ts
│   └── ...
├── routes/                     # Route definitions
│   ├── index.ts                # Root router: applies global middleware, mounts versioned routers
│   └── v1/
│       ├── index.ts            # V1 router: mounts all domain route files
│       ├── authRoutes.ts
│       ├── accountRoutes.ts
│       ├── workshopFactionRoutes.ts
│       └── ...
├── migrations/                 # sequelize-cli migration files (JavaScript)
├── templates/                  # EJS/HTML email templates
├── utils/
│   ├── enums.ts                # All enums for the project
│   ├── exceptions.ts           # Custom error classes
│   ├── middleware.ts            # Express middlewares (auth, validation, rate limit)
│   └── utility.ts              # Generic helper functions
└── tsconfig.json
```

### Naming Conventions

| File Type      | Pattern                                   | Example                            |
|----------------|-------------------------------------------|------------------------------------|
| Provider       | `{name}Provider.ts`                       | `hashProvider.ts`                  |
| Interface      | `I{Domain}.ts`                            | `IAccount.ts`                      |
| Schema         | `{domain}Schema.ts`                       | `accountSchema.ts`                 |
| Model          | `{domain}/{domainName}.model.ts`          | `account/account.model.ts`         |
| Service        | `{domain}Service.ts`                      | `accountService.ts`                |
| Orchestration  | `{domain}Orchestration.ts`                | `accountOrchestration.ts`          |
| Controller     | `{domain}Controller.ts`                   | `accountController.ts`             |
| Route          | `{domain}Routes.ts` (inside `routes/v1/`) | `accountRoutes.ts`                 |
| Migration      | `{timestamp}-{description}.js`            | `20260501070500-create-workshop-faction.js` |
| Enum           | All enums live in `utils/enums.ts`        | `EventTypeEnum`, `ItemTypeEnum`    |

---

## 3. Architecture Layers

The architecture follows a strict **6-layer** design with a clear dependency direction:

```
  Route → Controller → Orchestration → Service → Provider
                                              ↘ Model
```

### Layer Responsibilities

| Layer            | Responsibility                                                   | Depends On                    |
|------------------|------------------------------------------------------------------|-------------------------------|
| **Route**        | HTTP method + path + middleware chain                            | Controller, Middleware, Schema|
| **Controller**   | Parse `req`, call orchestration, format `res`, handle errors     | Orchestration (via DI index)  |
| **Orchestration**| Coordinate multiple services, manage transactions                | Service(s), Provider(s)       |
| **Service**      | Single-entity business logic, DB queries via models              | Provider(s), Model(s)         |
| **Provider**     | Wrap third-party libraries behind an interface                   | Third-party libraries         |
| **Model**        | Sequelize model definition (schema + table mapping)              | Database config               |

### Critical Rules

1. **Controllers NEVER call services directly** — they always go through orchestration.
2. **Services NEVER call other services** — cross-service coordination belongs in orchestration.
3. **Providers are stateless wrappers** — they abstract infrastructure concerns behind interfaces.
4. **Models are data-access only** — no business logic in model files.
5. **Each layer communicates through interfaces** — concrete classes implement interfaces.

---

## 4. Dependency Injection System

This project uses **manual constructor injection** with **composition-root index files**.
There is no DI container library. Wiring is explicit and centralized.

### How It Works

Each layer has an `index.ts` file that serves as its **composition root**:

```
provider/index.ts   → instantiates all providers (no dependencies)
services/index.ts   → imports providers, instantiates services with provider dependencies
orchestration/index.ts → imports services + providers, instantiates orchestrations
```

### The 3-File DI Chain

#### Step 1: `provider/index.ts` — Instantiate Providers

```typescript
// provider/index.ts
import { BcryptHashProvider } from "@/provider/hashProvider";
import { JwtProvider } from "@/provider/jwtProvider";
import { NodemailerEmailProvider } from "@/provider/emailProvider";
import { CloudflareR2StorageProvider } from "@/provider/storageProvider";

// Singleton instances — providers have no constructor dependencies
export const bcryptHashProvider = new BcryptHashProvider();
export const jwtProvider = new JwtProvider();
export const nodeMailerEmailProvider = new NodemailerEmailProvider();
export const cloudflareR2StorageProvider = new CloudflareR2StorageProvider();
```

#### Step 2: `services/index.ts` — Inject Providers into Services

```typescript
// services/index.ts
import { AccountService } from "@/services/accountService";
import { FileService } from "@/services/fileService";

import {
    bcryptHashProvider,
    validatorValidatorProvider,
    cloudflareR2StorageProvider,
} from "@/provider";

// Services receive their provider dependencies via constructor
export const accountService = new AccountService(
    bcryptHashProvider,
    validatorValidatorProvider,
);
export const fileService = new FileService(cloudflareR2StorageProvider);
```

#### Step 3: `orchestration/index.ts` — Inject Services into Orchestrations

```typescript
// orchestration/index.ts
import { AccountOrchestration } from "@/orchestration/accountOrchestration";

import {
    accountService,
    authService,
    notificationService,
} from "@/services";

// Orchestrations receive their service dependencies via constructor
export const accountOrchestration = new AccountOrchestration(
    accountService,
    authService,
    notificationService,
);
```

### Key DI Rules

1. **Constructor parameters are typed as interfaces**, not concrete classes.
2. **All dependencies flow downward**: Orchestration → Service → Provider.
3. **No circular dependencies** — if two services need each other, orchestration coordinates them.
4. **Wiring happens ONLY in `index.ts` files** — never in the consuming code.
5. **Controllers import from `orchestration/index.ts`** directly (they sit at the boundary).

---

## 5. Layer-by-Layer Guide with Examples

### 5.1 Provider Layer

Providers wrap third-party libraries behind a **TypeScript interface** so they can be
swapped without touching business logic.

**Rules:**
- Define the interface and the implementation in the **same file**.
- The interface is prefixed with `I` (e.g., `IHashProvider`).
- The class name includes the library name (e.g., `BcryptHashProvider`).
- Providers have **no business logic** — they are pure adapters.
- Constructor may read from `process.env` for configuration.

**Example: `hashProvider.ts`**

```typescript
import bcrypt from "bcrypt";

// interfaces
export interface IHashProvider {
    hashString(string: string, saltRounds?: number): Promise<string>;
    compareHash(plainText: string, hashed: string): Promise<boolean>;
}

export class BcryptHashProvider implements IHashProvider {
    async hashString(string: string, saltRounds: number = 10): Promise<string> {
        const salt = await bcrypt.genSalt(saltRounds);
        const hashedString = await bcrypt.hash(string, salt);
        return hashedString;
    }

    async compareHash(plainText: string, hashed: string): Promise<boolean> {
        const isMatch = await bcrypt.compare(plainText, hashed);
        return isMatch;
    }
}
```

**Example: `storageProvider.ts`**

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand, ... } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { GetPresignedUrlDTO } from '@/interfaces/IFile';

export interface IStorageProvider {
    getPresignedUrl(data: GetPresignedUrlDTO): Promise<string>;
    copyFile(sourceKey: string, destinationKey: string): Promise<void>;
    deleteFile(key: string): Promise<void>;
    // ... more methods
}

export class CloudflareR2StorageProvider implements IStorageProvider {
    private client: S3Client;
    private bucketName: string;

    constructor() {
        this.bucketName = process.env.R2_BUCKET_NAME || '';
        this.client = new S3Client({
            region: 'auto',
            endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
            credentials: {
                accessKeyId: process.env.R2_ACCESS_KEY_ID || '',
                secretAccessKey: process.env.R2_SECRET_ACCESS_KEY || '',
            },
        });
    }

    async getPresignedUrl(data: GetPresignedUrlDTO): Promise<string> {
        // ... implementation
    }
    // ... other methods
}
```

---

### 5.2 Interface Layer

Each domain has a dedicated `I{Domain}.ts` file containing:
- **DTOs** (`Create___DTO`, `Update___DTO`) for incoming data shapes.
- **Return types** (`___DataReturn`) for outgoing data shapes.

**Rules:**
- DTOs use `interface` keyword.
- Return types use `type` keyword.
- DTOs carry only the data needed for the operation — no model internals.
- Return types map model fields to camelCase (the service layer does the mapping).

**Example: `IFaction.ts`**

```typescript
export interface CreateFactionDTO {
    image?: string;
    name: string;
    description: string;
    color: string;
}

export interface UpdateFactionDTO {
    workshopFactionId: number;
    image?: string;
    isImageUpdated: boolean;
    name: string;
    description: string;
    color: string;
}

export type WorkshopFactionDataReturn = {
    workshopFactionId: number;
    accountId: string;
    image: string | null;
    name: string;
    description: string;
    color: string;
    createdAt: Date;
}
```

---

### 5.3 Schema Layer

Zod schemas validate `req.body` **before** it reaches the controller logic.
They are applied via the `validateRequest` middleware in the route definition.

**Rules:**
- One file per domain: `{domain}Schema.ts`.
- Schema names match the DTO names (e.g., `CreateFactionSchema` for `CreateFactionDTO`).
- Export each schema as a named `const`.
- Keep schemas flat and focused on validation — no transformation logic.

**Example: `factionSchema.ts`**

```typescript
import { z } from "zod";

export const CreateFactionSchema = z.object({
    image: z.string().optional(),
    name: z.string().min(1).max(100),
    description: z.string().min(1),
    color: z.string().min(1).max(50),
});

export const UpdateFactionSchema = z.object({
    workshopFactionId: z.number(),
    image: z.string().optional(),
    isImageUpdated: z.boolean(),
    name: z.string().min(1).max(100),
    description: z.string().min(1),
    color: z.string().min(1).max(50),
});
```

---

### 5.4 Model Layer

Models define Sequelize table schemas using `Model.init()`. Each model lives in its own
**subdirectory** under `models/`.

**Rules:**
- One model per file: `{domainName}.model.ts` inside `models/{domain}/`.
- Use `declare` for typed properties.
- Use `underscored: true` for column naming (snake_case in DB, declared as snake_case in TS).
- Always set `timestamps: true`.
- Export the model as `default`.
- Register in `models/index.ts` barrel export.

**Example: `workshopFaction/workshopFaction.model.ts`**

```typescript
import { Model, DataTypes } from "sequelize";
import sequelize from "@/config/database";

// models
import Account from "@/models/account/account.model";

class WorkshopFaction extends Model {
    declare public workshop_faction_id: number;
    declare public account_id: string;
    declare public image: string | null;
    declare public name: string;
    declare public description: string;
    declare public color: string;
    declare public readonly createdAt: Date;
    declare public readonly updatedAt: Date;
}

WorkshopFaction.init({
    workshop_faction_id: {
        type: DataTypes.INTEGER,
        autoIncrement: true,
        primaryKey: true,
        allowNull: false,
    },
    account_id: {
        type: DataTypes.UUIDV4,
        allowNull: false,
        references: {
            model: Account,
            key: 'account_id',
        },
        onDelete: 'CASCADE',
    },
    image: {
        type: DataTypes.STRING(100),
        allowNull: true,
    },
    name: {
        type: DataTypes.STRING(100),
        allowNull: false,
    },
    description: {
        type: DataTypes.TEXT,
        allowNull: false,
    },
    color: {
        type: DataTypes.STRING(50),
        allowNull: false,
    },
}, {
    sequelize,
    modelName: 'WorkshopFaction',
    tableName: 'workshop_factions',
    timestamps: true,
    underscored: true,
});

export default WorkshopFaction;
```

**Barrel export in `models/index.ts`:**

```typescript
import Account from "@/models/account/account.model";
import WorkshopFaction from "@/models/workshopFaction/workshopFaction.model";
// ... other models

export {
    Account,
    WorkshopFaction,
    // ...
}
```

---

### 5.5 Service Layer

Services contain **single-entity business logic**. They interact with models directly
and receive providers via constructor injection.

**Rules:**
- Define the service interface (`IXxxService`) and the class in the **same file**.
- The interface is exported alongside the class.
- Constructor takes **provider interfaces** as parameters (not concrete classes).
- Services **never call other services** — use orchestration for cross-service work.
- Services map model (snake_case) fields to camelCase return types.
- All methods have explicit return types.
- Throw custom exceptions (from `utils/exceptions.ts`) for error cases.
- Services that don't need providers still have an empty constructor: `constructor() { }`.

**Example: `workshopFactionService.ts`**

```typescript
import { Transaction } from "sequelize";

// models
import { WorkshopFaction } from "@/models";

// exceptions
import { DataNotFound } from "@/utils/exceptions";

// interfaces
import { CreateFactionDTO, UpdateFactionDTO, WorkshopFactionDataReturn } from "@/interfaces/IFaction";

export interface IWorkshopFactionService {
    getFactions(accountId: string): Promise<WorkshopFactionDataReturn[]>;
    getOneFaction(workshopFactionId: number, accountId: string, imageKeyOnly?: boolean): Promise<WorkshopFactionDataReturn>;
    createFaction(data: CreateFactionDTO, accountId: string, transaction?: Transaction): Promise<WorkshopFactionDataReturn>;
    editFaction(data: UpdateFactionDTO, accountId: string, transaction?: Transaction): Promise<WorkshopFactionDataReturn>;
    deleteFaction(workshopFactionId: number, accountId: string): Promise<void>;
    updateFactionImage(workshopFactionId: number, accountId: string, image: string): Promise<void>;
}

export class WorkshopFactionService implements IWorkshopFactionService {
    constructor() { }

    async getFactions(accountId: string): Promise<WorkshopFactionDataReturn[]> {
        const factions = await WorkshopFaction.findAll({
            where: { account_id: accountId },
            order: [['created_at', 'DESC']]
        });

        const imageBaseUrl = process.env.FILE_PUBLIC_URL || "";

        return factions.map(faction => ({
            workshopFactionId: faction.workshop_faction_id,
            accountId: faction.account_id,
            image: faction.image ? `${imageBaseUrl}/${faction.image}` : "",
            name: faction.name,
            description: faction.description,
            color: faction.color,
            createdAt: faction.createdAt,
        }));
    }

    async createFaction(data: CreateFactionDTO, accountId: string, transaction?: Transaction): Promise<WorkshopFactionDataReturn> {
        const newFaction = await WorkshopFaction.create({
            account_id: accountId,
            image: "",
            name: data.name,
            description: data.description,
            color: data.color,
        }, { transaction: transaction ?? null });

        return {
            workshopFactionId: newFaction.workshop_faction_id,
            accountId: newFaction.account_id,
            image: newFaction.image,
            name: newFaction.name,
            description: newFaction.description,
            color: newFaction.color,
            createdAt: newFaction.createdAt,
        };
    }

    async deleteFaction(workshopFactionId: number, accountId: string): Promise<void> {
        const faction = await WorkshopFaction.findOne({
            where: {
                workshop_faction_id: workshopFactionId,
                account_id: accountId,
            }
        });

        if (!faction) {
            throw new DataNotFound("FACTION001");
        }

        await faction.destroy();
    }

    // ... other methods follow the same pattern
}
```

**Service with Provider Dependencies: `accountService.ts`**

```typescript
// interfaces
import { IHashProvider } from "@/provider/hashProvider";
import { IValidatorProvider } from "@/provider/validatorProvider";

export interface IAccountService {
    createAccount(data: CreateAccountDTO): Promise<AccountDataReturn>;
    // ... other methods
}

export class AccountService implements IAccountService {
    constructor(
        private hashProvider: IHashProvider,
        private validatorProvider: IValidatorProvider,
    ) { }

    async createAccount(data: CreateAccountDTO): Promise<AccountDataReturn> {
        // Use injected providers — NEVER import concrete implementations
        const emailFormatValid = this.validatorProvider.validateEmail(data.email);
        if (!emailFormatValid) {
            throw new WrongFormat("Invalid email format");
        }

        const hashedPassword = await this.hashProvider.hashString(data.password);
        // ... rest of logic
    }
}
```

---

### 5.6 Orchestration Layer

Orchestrations coordinate **multiple services** and manage **database transactions**.
They are the only layer that can call multiple services together.

**Rules:**
- Define the orchestration interface (`IXxxOrchestration`) and class in the **same file**.
- Constructor takes **service interfaces** as parameters.
- Orchestration **owns transaction lifecycle** — `sequelize.transaction()`, `commit()`, `rollback()`.
- Orchestrations can also receive providers directly (e.g., event publishers).
- Methods that modify multiple tables MUST use transactions with try/catch/rollback.

**Example: `workshopFactionOrchestration.ts`**

```typescript
import sequelize from '@/config/database';

// interfaces
import { CreateFactionDTO, UpdateFactionDTO, WorkshopFactionDataReturn } from "@/interfaces/IFaction";
import { IFileService } from "@/services/fileService";
import { IWorkshopFactionService } from "@/services/workshopFactionService";

export interface IWorkshopFactionOrchestration {
    getFactions(accountId: string): Promise<WorkshopFactionDataReturn[]>;
    getOneFaction(workshopFactionId: number, accountId: string): Promise<WorkshopFactionDataReturn>;
    createFaction(data: CreateFactionDTO, accountId: string): Promise<WorkshopFactionDataReturn>;
    editFaction(data: UpdateFactionDTO, accountId: string): Promise<WorkshopFactionDataReturn>;
    deleteFaction(workshopFactionId: number, accountId: string): Promise<void>;
}

export class WorkshopFactionOrchestration implements IWorkshopFactionOrchestration {
    constructor(
        private factionService: IWorkshopFactionService,
        private fileService: IFileService,
    ) { }

    async getFactions(accountId: string): Promise<WorkshopFactionDataReturn[]> {
        return await this.factionService.getFactions(accountId);
    }

    async createFaction(data: CreateFactionDTO, accountId: string): Promise<WorkshopFactionDataReturn> {
        const transaction = await sequelize.transaction();

        try {
            const newFaction = await this.factionService.createFaction(data, accountId, transaction);
            const imageKey = await this.fileService.moveTempFileToFinalLocation(
                data.image ?? "",
                `user/uploads/workshop/faction/${newFaction.workshopFactionId}-${accountId}-${Date.now()}`
            );

            await transaction.commit();

            await this.factionService.updateFactionImage(newFaction.workshopFactionId, accountId, imageKey);

            return await this.factionService.getOneFaction(newFaction.workshopFactionId, accountId);

        } catch (error) {
            await transaction.rollback();
            throw error;
        }
    }

    async deleteFaction(workshopFactionId: number, accountId: string): Promise<void> {
        const faction = await this.factionService.getOneFaction(workshopFactionId, accountId, true);

        if (faction.image) {
            await this.fileService.deleteFile(faction.image);
        }

        await this.factionService.deleteFaction(workshopFactionId, accountId);
    }
}
```

---

### 5.7 Controller Layer

Controllers sit at the HTTP boundary. They parse the request, call orchestration,
and format the response.

**Rules:**
- Controllers are `class` with `static async` methods — no instantiation needed.
- Import orchestrations from `@/orchestration` (the composition root).
- Each method follows the pattern: **try → call orchestration → send JSON → catch → send error JSON**.
- Controllers **do not contain business logic**.
- Authenticated user data comes from `req.user?.accountId`.
- Use `instanceof` checks against custom exception classes for error mapping.
- Export as `default`.

**Example: `workshopFactionController.ts`**

```typescript
import { Request, Response } from 'express';

// exceptions
import { WrongFormat, DataNotFound } from '@/utils/exceptions';

// orchestration
import { factionOrchestration } from '@/orchestration';

class WorkshopFactionController {

    static async getFactions(req: Request, res: Response) {
        try {
            const accountId = req.user?.accountId || "";
            const data = await factionOrchestration.getFactions(accountId);

            return res.status(200).json({
                "status": "success",
                "message": "Factions fetched successfully",
                "userMessage": "",
                "data": data
            });

        } catch (e) {
            return res.status(500).json({
                "status": "failed",
                "message": "Internal server error",
                "userMessage": "500",
                "errors": e
            });
        }
    }

    static async createFaction(req: Request, res: Response) {
        try {
            const accountId = req.user?.accountId || "";
            const data = await factionOrchestration.createFaction(req.body, accountId);

            return res.status(201).json({
                "status": "success",
                "message": "Faction created successfully",
                "userMessage": "",
                "data": data
            });

        } catch (e) {
            if (e instanceof WrongFormat) {
                return res.status(422).json({
                    "status": "failed",
                    "message": e.message,
                    "userMessage": "",
                });
            }

            return res.status(500).json({
                "status": "failed",
                "message": "Internal server error",
                "userMessage": "500",
                "errors": e
            });
        }
    }

    static async deleteFaction(req: Request, res: Response) {
        try {
            const accountId = req.user?.accountId || "";
            await factionOrchestration.deleteFaction(Number(req.params.id), accountId);

            return res.status(200).json({
                "status": "success",
                "message": "Faction deleted successfully",
                "userMessage": "",
            });

        } catch (e) {
            if (e instanceof DataNotFound) {
                return res.status(404).json({
                    "status": "failed",
                    "message": "Faction not found",
                    "userMessage": e.message,
                });
            }

            return res.status(500).json({
                "status": "failed",
                "message": "Internal server error",
                "userMessage": "500",
                "errors": e
            });
        }
    }
}

export default WorkshopFactionController;
```

---

### 5.8 Route Layer

Routes wire HTTP paths to middleware chains and controller methods.

**Rules:**
- Each domain has its own route file in `routes/v1/`.
- Routes use `Router()` from Express.
- Apply `authenticate` middleware for protected routes.
- Apply `validateRequest(Schema)` for routes that accept a body.
- Middleware order: `authenticate` → `validateRequest` → controller method.
- Export as `default`.
- Register in `routes/v1/index.ts`.

**Example: `workshopFactionRoutes.ts`**

```typescript
import { Router } from 'express';

// middlewares
import { authenticate, validateRequest } from '@/utils/middleware';

// schema
import { CreateFactionSchema, UpdateFactionSchema } from '@/schema/factionSchema';

// controller
import WorkshopFactionController from '@/controller/workshopFactionController';

const workshopFactionRoutes = Router();

workshopFactionRoutes.get(
    '/',
    authenticate,
    WorkshopFactionController.getFactions,
);
workshopFactionRoutes.get(
    '/:id',
    authenticate,
    WorkshopFactionController.getOneFaction,
);
workshopFactionRoutes.post(
    '/',
    authenticate,
    validateRequest(CreateFactionSchema),
    WorkshopFactionController.createFaction,
);
workshopFactionRoutes.put(
    '/',
    authenticate,
    validateRequest(UpdateFactionSchema),
    WorkshopFactionController.editFaction,
);
workshopFactionRoutes.delete(
    '/:id',
    authenticate,
    WorkshopFactionController.deleteFaction,
);

export default workshopFactionRoutes;
```

**Register in `routes/v1/index.ts`:**

```typescript
import { Router } from 'express';
import workshopFactionRoutes from './workshopFactionRoutes';

const v1Api = Router();
v1Api.use("/workshop-faction", workshopFactionRoutes);
// ... other routes

export default v1Api;
```

---

## 6. Wiring a New Feature End-to-End

When adding a new domain entity (e.g., "Workshop Skill"), follow this exact order:

### Step 1: Interface — `interfaces/ISkill.ts`
Define `CreateSkillDTO`, `UpdateSkillDTO`, and `WorkshopSkillDataReturn`.

### Step 2: Schema — `schema/skillSchema.ts`
Define `CreateSkillSchema` and `UpdateSkillSchema` with Zod.

### Step 3: Model — `models/workshopSkill/workshopSkill.model.ts`
Define the Sequelize model. Register it in `models/index.ts`.

### Step 4: Migration — `migrations/{timestamp}-create-workshop-skill.js`
Create a JavaScript migration file.

### Step 5: Service — `services/workshopSkillService.ts`
Define `IWorkshopSkillService` and `WorkshopSkillService`.
Register in `services/index.ts`:
```typescript
export const workshopSkillService = new WorkshopSkillService();
```

### Step 6: Orchestration — `orchestration/workshopSkillOrchestration.ts`
Define `IWorkshopSkillOrchestration` and `WorkshopSkillOrchestration`.
Register in `orchestration/index.ts`:
```typescript
export const skillOrchestration = new WorkshopSkillOrchestration(
    workshopSkillService,
    fileService,
);
```

### Step 7: Controller — `controller/workshopSkillController.ts`
Create the controller class with static methods.

### Step 8: Route — `routes/v1/workshopSkillRoutes.ts`
Wire routes with middleware. Register in `routes/v1/index.ts`:
```typescript
v1Api.use("/workshop-skill", workshopSkillRoutes);
```

---

## 7. Supporting Infrastructure

### Custom Exceptions (`utils/exceptions.ts`)

```typescript
export class ExistData extends Error {
    constructor(message: string) {
        super(message);
        this.name = "ExistDataError";
    }
}

export class DataNotFound extends Error { ... }
export class WrongFormat extends Error { ... }
export class NotValid extends Error { ... }
export class Forbidden extends Error { ... }
```

**Exception → HTTP Status Mapping:**

| Exception      | HTTP Status | Usage                         |
|----------------|-------------|-------------------------------|
| `DataNotFound` | 404         | Entity not found              |
| `ExistData`    | 409         | Duplicate/conflict            |
| `WrongFormat`  | 422         | Validation/format error       |
| `NotValid`     | 401         | Token/auth invalid            |
| `Forbidden`    | 403         | Permission denied / rate limit|

### Middleware (`utils/middleware.ts`)

- **`validateRequest(Schema)`** — Zod schema validation middleware. Returns 400 with `schemaErrors` on failure.
- **`authenticate`** — Verifies JWT from `Authorization: Bearer <token>` header. Sets `req.user`.
- **`apiRateLimiter`** — Global rate limiting (30 req/sec per IP).

### Global Type Augmentation (`config/index.d.ts`)

```typescript
import { AccessTokenBody } from "@/interfaces/IAuth";

declare global {
    namespace Express {
        interface Request {
            user?: AccessTokenBody;
        }
    }
}

export { };
```

### Enums (`utils/enums.ts`)

All enums are centralized in a single file. Use enums for any fixed string comparisons:

```typescript
export enum EventTypeEnum {
    OTP_GENERATED = "otpGenerated",
}

export enum ItemTypeEnum {
    GEAR = "gear",
    WEAPON = "weapon",
    ARMOR = "armor",
}
```

---

## 8. API Response Format

All API responses follow a consistent JSON shape:

### Success Response

```json
{
    "status": "success",
    "message": "Descriptive success message",
    "userMessage": "",
    "data": { ... }
}
```

### Error Response

```json
{
    "status": "failed",
    "message": "Descriptive error message",
    "userMessage": "ERROR_CODE or empty string",
    "errors": ...
}
```

### Validation Error Response (400)

```json
{
    "status": "failed",
    "message": "bad request",
    "userMessage": "",
    "schemaErrors": [
        { "field": "name", "message": "Required" }
    ]
}
```

**Key conventions:**
- `status`: always `"success"` or `"failed"`.
- `message`: human-readable description for developers.
- `userMessage`: client-facing error code or empty string. Uses error codes like `"AUTH001"`, `"ACCOUNT002"`, `"FACTION001"`.
- `data`: the response payload (omit on errors without data).
- Create responses use HTTP `201`.
- Delete/update responses use HTTP `200`.
- Internal server errors always return `"userMessage": "500"`.

---

## 9. Error Handling Pattern

Controllers use a consistent try/catch pattern with `instanceof` checks:

```typescript
static async someAction(req: Request, res: Response) {
    try {
        // 1. Extract accountId from auth
        const accountId = req.user?.accountId || "";

        // 2. Call orchestration
        const data = await someOrchestration.doSomething(req.body, accountId);

        // 3. Return success
        return res.status(200).json({
            "status": "success",
            "message": "Action completed",
            "userMessage": "",
            "data": data
        });

    } catch (e) {
        // 4. Handle known exceptions with specific status codes
        if (e instanceof DataNotFound) {
            return res.status(404).json({
                "status": "failed",
                "message": "Resource not found",
                "userMessage": e.message,
            });
        }

        if (e instanceof WrongFormat) {
            return res.status(422).json({
                "status": "failed",
                "message": e.message,
                "userMessage": "",
            });
        }

        // 5. Catch-all for unexpected errors
        return res.status(500).json({
            "status": "failed",
            "message": "Internal server error",
            "userMessage": "500",
            "errors": e
        });
    }
}
```

---

## 10. Migration Pattern

Migrations use **JavaScript** (required by `sequelize-cli`) and follow this structure:

```javascript
'use strict';

/** @type {import('sequelize-cli').Migration} */
module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.createTable('workshop_factions', {
      workshop_faction_id: {
        type: Sequelize.INTEGER,
        autoIncrement: true,
        primaryKey: true,
        allowNull: false,
      },
      account_id: {
        type: Sequelize.UUID,
        allowNull: false,
        references: {
          model: 'accounts',
          key: 'account_id',
        },
        onDelete: 'CASCADE'
      },
      // ... columns
      created_at: {
        type: Sequelize.DATE,
        allowNull: false,
        defaultValue: Sequelize.fn('NOW'),
      },
      updated_at: {
        type: Sequelize.DATE,
        allowNull: false,
        defaultValue: Sequelize.fn('NOW'),
      }
    });
  },

  async down(queryInterface, Sequelize) {
    await queryInterface.dropTable('workshop_factions');
  }
};
```

**Rules:**
- File name format: `{YYYYMMDDHHMMSS}-{description}.js`
- Always include `created_at` and `updated_at` with `defaultValue: Sequelize.fn('NOW')`.
- Use `references` for foreign keys with `onDelete: 'CASCADE'`.
- All column names use snake_case.
- The `down` method must reverse the `up` method.

---

## 11. Checklist for New Features

Before considering a backend feature complete, verify:

- [ ] **Interface file** created in `interfaces/` with DTOs and return types
- [ ] **Zod schema** created in `schema/` matching the DTOs
- [ ] **Model** defined in `models/{domain}/` and registered in `models/index.ts`
- [ ] **Migration** created in `migrations/` (JavaScript)
- [ ] **Service** defined with interface + class, registered in `services/index.ts`
- [ ] **Orchestration** defined with interface + class, registered in `orchestration/index.ts`
- [ ] **Controller** created with static methods, error handling follows the pattern
- [ ] **Routes** created in `routes/v1/`, registered in `routes/v1/index.ts`
- [ ] **Middleware** applied correctly: `authenticate` for protected, `validateRequest` for body validation
- [ ] **DI wiring** done in all 3 composition roots (`provider/`, `services/`, `orchestration/` index files)
- [ ] **No service-to-service calls** — all cross-service logic lives in orchestration
- [ ] **Transactions** used for multi-table mutations with proper try/catch/rollback
- [ ] **Error handling** uses custom exceptions with correct HTTP status mapping
- [ ] **Enums** used instead of magic strings (added to `utils/enums.ts`)
- [ ] **API response format** follows the standard `{ status, message, userMessage, data }` shape

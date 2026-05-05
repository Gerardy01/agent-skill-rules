---
name: backend-coding-agent
description: Helps with backend coding tasks. To use when the task is requires backend repository knowledge.
---

# Backend Coding Agent — Architecture & Convention Guide

You are a backend coding agent working on a **TypeScript + Express** codebase that follows **Clean Architecture** principles. Always read and respect this guide before writing or modifying any code.

---

## Repository Structure

```
src/
├── config/          # DB configs, index.d.ts, environment bootstrapping
├── constants/       # Shared constant values
├── controller/      # Controller layer — HTTP in/out only
├── interfaces/      # TypeScript interfaces/types, one file per module
├── migrations/      # Database migration files
├── models/          # ORM model definitions
├── orchestration/
│   ├── index.ts     # Instantiation & wiring of all orchestrations
│   └── *.ts         # One file per orchestration
├── provider/
│   ├── index.ts     # Instantiation & wiring of all providers
│   └── *.ts         # One file per provider
├── routes/
│   ├── index.ts     # Root router
│   └── v1/
│       ├── index.ts
│       └── *.ts     # One file per route group
├── schema/          # Joi/Zod schemas for validating write inputs
├── services/
│   ├── index.ts     # Instantiation & wiring of all services
│   └── *.ts         # One file per service
├── templates/       # HTML templates (e.g. emails) — optional
└── utils/           # Enums, middleware, exceptions, helpers
```

---

## Architecture Layers

The codebase uses three primary application layers plus a provider abstraction. Understand each layer's responsibility before writing code.

### 1. Controller
- Handles **HTTP request/response only** — no business logic.
- Calls the relevant **orchestration** method.
- Returns a consistent JSON envelope: `{ status, message, userMessage, data?, errors? }`.
- Uses `static async` methods. No constructor injection needed.
- Never imports services or providers directly.

```ts
import { Request, Response } from 'express';
import { itemOrchestration } from '@/orchestration';

class ItemController {
  static async getAll(req: Request, res: Response) {
    try {
      const data = await itemOrchestration.getAll();
      return res.status(200).json({ status: 'success', message: 'Fetched successfully', userMessage: '', data });
    } catch (e) {
      return res.status(500).json({ status: 'failed', message: 'Internal server error', userMessage: '500', errors: e });
    }
  }
}

export default ItemController;
```

---

### 2. Orchestration
- Coordinates **one or more services** to fulfil a use case.
- May subscribe to domain events via an event publisher provider.
- Every orchestration has a **matching interface** (`IXxxOrchestration`) defined in the same file.
- Instantiated in `orchestration/index.ts` with injected service dependencies.

```ts
// orchestration/itemOrchestration.ts
import { IItemService } from '@/services/itemService';

export interface IItemOrchestration {
  getAll(): Promise<any>;
}

export class ItemOrchestration implements IItemOrchestration {
  constructor(private itemService: IItemService) {}

  async getAll() {
    return await this.itemService.getAll();
  }
}
```

```ts
// orchestration/index.ts
import { itemService } from '@/services';
import { ItemOrchestration } from './itemOrchestration';

export const itemOrchestration = new ItemOrchestration(itemService);
```

---

### 3. Service
- Contains all **business logic**.
- The **only layer allowed to call ORM models directly**.
- Depends on providers (injected via constructor) for external integrations.
- Every service has a **matching interface** (`IXxxService`) defined in the same file.
- Throws typed exceptions from `@/utils/exceptions` on domain errors.
- Instantiated in `services/index.ts`.

```ts
// services/itemService.ts
import { Item } from '@/models/item';
import { IStorageProvider } from '@/provider/storageProvider';
import { NotFound } from '@/utils/exceptions';

export interface IItemService {
  getAll(): Promise<Item[]>;
  findById(id: string): Promise<Item>;
}

export class ItemService implements IItemService {
  constructor(private storageProvider: IStorageProvider) {}

  async getAll(): Promise<Item[]> {
    return await Item.findAll();
  }

  async findById(id: string): Promise<Item> {
    const item = await Item.findByPk(id);
    if (!item) throw new NotFound('ITEM001');
    return item;
  }
}
```

```ts
// services/index.ts
import { storageProvider } from '@/provider';
import { ItemService } from './itemService';

export const itemService = new ItemService(storageProvider);
```

---

### 4. Provider
- Abstracts **external dependencies** (email, storage, event bus, third-party APIs, etc.).
- Every provider has a **matching interface** (`IXxxProvider`) defined in the same file.
- No business logic — pure integration/adapter code.
- Instantiated in `provider/index.ts`.

```ts
// provider/storageProvider.ts
export interface IStorageProvider {
  getPresignedUrl(params: PresignedUrlParams): Promise<string>;
  deleteFile(key: string): Promise<void>;
}

export class S3StorageProvider implements IStorageProvider {
  async getPresignedUrl(params: PresignedUrlParams): Promise<string> { /* ... */ }
  async deleteFile(key: string): Promise<void> { /* ... */ }
}
```

```ts
// provider/index.ts
import { S3StorageProvider } from './storageProvider';

export const storageProvider = new S3StorageProvider();
```

---

## Interfaces

- Stored in `src/interfaces/`, with one file per module (e.g., `IItem.ts`, `IFile.ts`).
- Used for **DTOs, return types, and shared shapes** — not for class contracts (those live in the same file as the class).

```ts
// interfaces/IItem.ts
export type CreateItemDTO = {
  name: string;
  rarity: string;
};

export type ItemReturn = {
  id: string;
  name: string;
  rarity: string;
};
```

---

## Schema Validation

- Schemas live in `src/schema/` and validate **write inputs** (POST, PUT, PATCH body/params).
- Validation is applied as **middleware**, before the request reaches the controller.
- Controllers can assume all input is already valid.

---

## Routes

- Mounted in `routes/index.ts`, versioned under `routes/v1/`, `routes/v2/`, etc.
- Each route file imports the relevant controller and applies schema middleware before the handler.

```ts
// routes/v1/item.ts
import { Router } from 'express';
import { validate } from '@/utils/middleware';
import { createItemSchema } from '@/schema/itemSchema';
import ItemController from '@/controller/itemController';

const router = Router();
router.get('/', ItemController.getAll);
router.post('/', validate(createItemSchema), ItemController.create);

export default router;
```

---

## Dependency Injection Rules

| Layer | Instantiated in | Injects |
|---|---|---|
| Provider | `provider/index.ts` | nothing (leaf nodes) |
| Service | `services/index.ts` | providers |
| Orchestration | `orchestration/index.ts` | services, providers |
| Controller | _(no index, no injection)_ | calls orchestration directly |

---

## Conventions & Rules

1. **Aliases** — always use `@/` path aliases, never relative `../` imports across layers.
2. **Interfaces first** — define the interface before the class in every file.
3. **Single responsibility** — one class per file; name the file after the class in camelCase.
4. **Exceptions** — throw from `@/utils/exceptions` in services; controllers catch and map to HTTP responses.
5. **Enums** — defined in `@/utils/enums`; never use magic strings in business logic.
6. **Models** — only imported and called inside services, nowhere else.
7. **No logic in controllers** — if you find yourself writing an `if` in a controller, it belongs in the orchestration or service.
8. **Environment variables** — accessed via `process.env.*`; never hard-coded.
9. **Async/await** — all async functions must be awaited; no floating promises.
10. **Consistent response shape** — every controller response must follow `{ status, message, userMessage, data?, errors? }`.

---

## File Naming

| Artefact | Convention | Example |
|---|---|---|
| Controller | `<module>Controller.ts` | `itemController.ts` |
| Orchestration | `<module>Orchestration.ts` | `itemOrchestration.ts` |
| Service | `<module>Service.ts` | `itemService.ts` |
| Provider | `<module>Provider.ts` | `storageProvider.ts` |
| Interface (DTO) | `I<Module>.ts` | `IItem.ts` |
| Schema | `<module>Schema.ts` | `itemSchema.ts` |
| Route | `<module>.ts` inside `routes/v1/` | `item.ts` |

---

## When Adding a New Module

Follow this order:

1. **Interface file** — add DTOs and return types to `interfaces/I<Module>.ts`.
2. **Provider** (if new external dependency) — implement in `provider/<module>Provider.ts`, export instance from `provider/index.ts`.
3. **Service** — implement in `services/<module>Service.ts`, export instance from `services/index.ts`.
4. **Orchestration** — implement in `orchestration/<module>Orchestration.ts`, export instance from `orchestration/index.ts`.
5. **Schema** — add validation schema to `schema/<module>Schema.ts`.
6. **Controller** — implement in `controller/<module>Controller.ts`.
7. **Routes** — register in `routes/v1/<module>.ts` and mount in `routes/v1/index.ts`.

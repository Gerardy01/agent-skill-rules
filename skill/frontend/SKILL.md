---
name: frontend-coding-agent
description: Use this skill to develop the frontend code, you must follow the rules in the document.
---

# Frontend Coding Agent — Architecture & Convention Guide

You are a frontend coding agent working on a **TypeScript + React + Vite** codebase. Always read and respect this guide before writing or modifying any code.

---

## Repository Structure

```
src/
├── api/                  # Encapsulated API request modules
│   ├── index.ts          # Re-exports all API class instances
│   ├── <module1>Api.ts
│   └── <module2>Api.ts
├── assets/               # Static assets (icons, images, fonts)
│   ├── index.ts          # Re-exports all assets
│   └── ...
├── components/           # Reusable UI components
│   ├── global/           # Shared across the whole app
│   └── <feature>/        # Feature-scoped components
├── config/               # App configuration files
│   │                     # e.g. axiosConfig.ts, i18n.d.ts
├── constants/            # Constant values
│   │                     # e.g. locales, theme, selections
├── hooks/                # Custom React hooks (all logic lives here)
│   └── <feature>/
│       └── use<Name>.ts
├── models/               # TypeScript interfaces and types
│   ├── globalInterfaces.ts
│   └── <module>Interfaces.ts
├── pages/                # Page-level components (route targets)
├── stores/               # Global state (e.g. Zustand, Redux)
├── utils/                # Utility functions, enums, helpers
├── App.tsx
└── main.tsx
```

---

## Core Rules

### Rule 1 — TSX Component Format

Every `.tsx` file must follow this exact structure:

1. Imports (see Rule 5 for order)
2. `Props` interface (if applicable)
3. Default exported function component
4. `styles` object at the bottom

```tsx
import { Button, Typography } from "antd";

// hooks
import useItemList from "@/hooks/item/useItemList";

// interfaces
import type { ItemReturn } from "@/models/itemInterfaces";

interface Props {
    onSelect: (item: ItemReturn) => void;
}

export default function ItemList({ onSelect }: Props) {
    const { items, isLoading } = useItemList();

    return (
        <div style={styles.container}>
            <Typography.Title level={4}>Items</Typography.Title>
        </div>
    );
}

const styles: { [key: string]: React.CSSProperties } = {
    container: {
        width: '100%',
        padding: '16px',
    },
};
```

---

### Rule 2 — API Module Format

- One class per file, one file per backend resource.
- Every method returns a **tuple**: `[undefined, Data]` on success, `[ErrorResponse]` on failure.
- Always use `catchFetchError` to wrap axios calls — never try/catch directly.
- Use `axiosPrivate` for authenticated routes, `axiosPublic` for open routes.
- Instantiate and export from `api/index.ts`.

```ts
// api/itemApi.ts
import { axiosPrivate } from "@/config/axiosConfig";

// utils
import { catchFetchError } from "@/utils/utility";

// interfaces
import type { FetchResponse, ErrorResponse } from "@/models/globalInterfaces";
import type { CreateItemDTO, ItemReturn } from "@/models/itemInterfaces";

export class ItemApi {
    async getItems(): Promise<[undefined, ItemReturn[]] | [ErrorResponse]> {
        const [error, res] = await catchFetchError(
            axiosPrivate.get<FetchResponse<ItemReturn[]>>('/item')
        );
        if (error) return [error];
        return [undefined, res.data.data];
    }

    async createItem(data: CreateItemDTO): Promise<[undefined, ItemReturn] | [ErrorResponse]> {
        const [error, res] = await catchFetchError(
            axiosPrivate.post<FetchResponse<ItemReturn>>('/item', data)
        );
        if (error) return [error];
        return [undefined, res.data.data];
    }
}
```

```ts
// api/index.ts
import { ItemApi } from "./itemApi";
import { WorkshopClassApi } from "./workshopClassApi";

export const itemApi = new ItemApi();
export const workshopClassApi = new WorkshopClassApi();
```

#### Global Response Interfaces

These are fixed contracts — do not modify or redefine them elsewhere.

```ts
// models/globalInterfaces.ts

interface BaseResponse {
    status: string;
    message: string;
    userMessage: string;
}

export interface FetchResponse<T> extends BaseResponse {
    data: T;
}

interface SchemaErrors {
    field: string;
    message: string;
}

interface BaseResponseError extends BaseResponse {
    schemaErrors?: SchemaErrors[];
}

export interface ErrorResponse {
    status: number;
    response: {
        data: BaseResponseError;
    };
}
```

---

### Rule 3 — Separate UI and Logic

**Components are for UI only. Hooks are for logic.**

- **Never** declare state (`useState`, `useReducer`) directly inside a component.
- **Never** declare functions, effects, or refs directly inside a component.
- Always extract all logic into a dedicated custom hook named `use<ComponentName>`.
- The component only calls the hook and renders the returned values.
- Still allowed to import trnaslation related hooks directly inside components

```ts
// hooks/item/useItemList.ts
import { useState, useEffect } from "react";

// api
import { itemApi } from "@/api";

// interfaces
import type { ItemReturn } from "@/models/itemInterfaces";

export default function useItemList() {
    const [items, setItems] = useState<ItemReturn[]>([]);
    const [isLoading, setIsLoading] = useState(false);

    useEffect(() => {
        const fetch = async () => {
            setIsLoading(true);
            const [error, data] = await itemApi.getItems();
            if (!error) setItems(data);
            setIsLoading(false);
        };
        fetch();
    }, []);

    return { items, isLoading };
}
```

```tsx
// components/item/ItemList.tsx

// hooks
import useItemList from "@/hooks/item/useItemList";

export default function ItemList() {
    const { items, isLoading } = useItemList();

    return (
        <div style={styles.container}>
            {/* render items */}
        </div>
    );
}

const styles: { [key: string]: React.CSSProperties } = {
    container: { width: '100%' },
};
```

---

### Rule 4 — One Component Per File

- **Never** define two or more components in a single `.tsx` file.
- If a UI piece needs to be broken down, create a new file for each sub-component.
- Small helper render functions are allowed only if they return a primitive JSX snippet (not a standalone component with its own props or state). When in doubt, split into a new file.

---

### Rule 5 — Import Order

Imports must follow this exact order, with a blank line between each group and a comment prefix for all internal groups:

```tsx
// 1. Third-party libraries (no comment prefix)
import { Button, Form, Input, Modal, Select, Typography } from "antd";
import { useTranslation } from "react-i18next";

// utils
import { ItemTypeEnum } from "@/utils/enums";
import { formatDate } from "@/utils/utility";

// hooks
import useCreateItem from "@/hooks/item/useCreateItem";

// assets
import { SparklesIcon } from "@/assets";
import { DeleteOutlined, PlusOutlined } from "@ant-design/icons";

// components
import ImageForm from "@/components/global/form/ImageForm";

// interfaces
import type { CreateItemDTO } from "@/models/itemInterfaces";

// 7. Local Props interface — defined inline below imports, not imported
interface Props {
    open: boolean;
    onClose: () => void;
}
```

Rules within import groups:
- Always use `import type` for TypeScript interfaces and types.
- Third-party imports carry **no comment prefix**; all internal imports use their labeled group comment.
- `@/` alias is mandatory for all internal imports — never use relative `../` paths across feature folders.

# Addendum — Internationalisation (i18n) Rules

---

### Rule 6 — Always Use Translations for Text

**Never hard-code display text in components or hooks.** Every user-facing string must be defined in a locale file and accessed via the `useTranslation` hook.

#### Locale Files (`src/constants/locales/`)

- One JSON file per language: `en.json`, `id.json`, etc.
- Keys are organised by feature/module namespace using nested objects.
- Key names use `camelCase`. Namespace names use `camelCase`.
- All locale files must have the **same key structure** — never add a key to one language without adding it to all others.

```
src/constants/locales/
├── en.json
├── id.json
└── ...
```

```json
// constants/locales/en.json
{
    "common": {
        "save": "Save",
        "cancel": "Cancel",
        "delete": "Delete",
        "confirm": "Are you sure?"
    },
    "item": {
        "title": "Items",
        "createTitle": "Create Item",
        "fields": {
            "name": "Name",
            "rarity": "Rarity"
        },
        "messages": {
            "createSuccess": "Item created successfully",
            "deleteSuccess": "Item deleted successfully"
        }
    }
}
```

```json
// constants/locales/id.json
{
    "common": {
        "save": "Simpan",
        "cancel": "Batal",
        "delete": "Hapus",
        "confirm": "Apakah Anda yakin?"
    },
    "item": {
        "title": "Item",
        "createTitle": "Buat Item",
        "fields": {
            "name": "Nama",
            "rarity": "Kelangkaan"
        },
        "messages": {
            "createSuccess": "Item berhasil dibuat",
            "deleteSuccess": "Item berhasil dihapus"
        }
    }
}
```

---

#### Using Translations in Components

- Call `useTranslation` inside the **components** or **hook**.
- Import `useTranslation` in the third-party group (no comment prefix) at the top of the file.
- Use **dot notation** with the namespace as the first segment: `t('item.title')`.

```tsx
// components/item/ItemList.tsx
import { Typography } from "antd";
import { useTranslation } from "react-i18next";

// hooks
import useItemList from "@/hooks/item/useItemList";

export default function ItemList() {
    const { t } = useTranslation();
    const { items, isLoading } = useItemList();

    return (
        <div style={styles.container}>
            <Typography.Title level={4}>{t('item.title')}</Typography.Title>
        </div>
    );
}

const styles: { [key: string]: React.CSSProperties } = {
    container: { width: '100%' },
};
```

#### Using Translations in Hooks (for messages, alerts, etc.)

When a hook needs to produce a translated string (e.g. a toast message or a confirm dialog text), `useTranslation` is called inside the hook and `t` is used there directly.

```ts
// hooks/item/useDeleteItem.ts
import { useTranslation } from "react-i18next";

// api
import { itemApi } from "@/api";

export default function useDeleteItem() {
    const { t } = useTranslation();

    const deleteItem = async (id: string) => {
        const [error] = await itemApi.deleteItem(id);
        if (error) return;
        alert(t('item.messages.deleteSuccess'));
    };

    return { deleteItem };
}
```

#### Key Naming Rules

| Situation | Key path pattern | Example |
|---|---|---|
| Page / section title | `<module>.title` | `item.title` |
| Modal / drawer title | `<module>.<action>Title` | `item.createTitle` |
| Form field label | `<module>.fields.<fieldName>` | `item.fields.name` |
| Success / error message | `<module>.messages.<name>` | `item.messages.createSuccess` |
| Shared UI text | `common.<name>` | `common.save` |

#### Hard Rules

1. **No raw strings in JSX** — every string rendered to the user must go through `t()`.
2. **No inline fallbacks** — never write `t('some.key') || 'fallback text'`; if a key is missing, fix the locale file.
3. **Add to all locales simultaneously** — adding a key to `en.json` without adding it to `id.json` (and any other languages) is not allowed.

---

## Additional Conventions

### Models (`src/models/`)
- One file per domain module: `itemInterfaces.ts`, `classInterfaces.ts`, etc.
- `globalInterfaces.ts` holds shared response shapes (`FetchResponse`, `ErrorResponse`, etc.) — never redefine these.
- Use `type` for simple shapes and union types; use `interface` for objects that may be extended.

```ts
// models/itemInterfaces.ts
export interface ItemReturn {
    id: string;
    name: string;
    rarity: string;
}

export type CreateItemDTO = {
    name: string;
    rarity: string;
};
```

### Hooks (`src/hooks/`)
- Organised by feature folder: `hooks/<feature>/use<Name>.ts`.
- Hook filename and exported function name must match exactly: `useCreateItem.ts` exports `useCreateItem`.
- A hook tied to a single component is named after that component: `useItemFormModal.ts` for `ItemFormModal.tsx`.
- Hooks shared across features live at the root of `hooks/` (e.g., `hooks/useDebounce.ts`).

### Stores (`src/stores/`)
- Only truly **global** state lives here (auth session, theme, current user).
- Feature-scoped state belongs in the feature hook, not the store.

### Constants (`src/constants/`)
- Theme tokens, dropdown selections, locale keys, and static lookup tables.
- Never hard-code a value used in more than one place — extract it here.

### Enums (`src/utils/enums.ts`)
- All enums are defined in this single file.
- Never use magic strings in components or hooks — always reference an enum.

```ts
export enum ItemTypeEnum {
    WEAPON = 'weapon',
    ARMOR = 'armor',
    CONSUMABLE = 'consumable',
}
```

### Assets (`src/assets/`)
- All asset imports go through `assets/index.ts` — never import an asset file directly by its path in a component.

---

## When Adding a New Feature

Follow this order:

1. **Models** — add DTOs and return types to `models/<module>Interfaces.ts`.
2. **API module** — implement `api/<module>Api.ts`; export the instance from `api/index.ts`.
3. **Hooks** — implement logic in `hooks/<feature>/use<Name>.ts`.
4. **Components** — build UI in `components/<feature>/<ComponentName>.tsx`, consuming only the hook.
5. **Page** — assemble the page in `pages/<PageName>.tsx`, importing feature components.
6. **Routes** — register the new page in the router inside `App.tsx` or the routing config.

---

## Quick Reference — What Goes Where

| Concern | Location |
|---|---|
| API call logic | `api/<module>Api.ts` |
| TypeScript types & DTOs | `models/<module>Interfaces.ts` |
| Business / UI logic, state, effects | `hooks/<feature>/use<Name>.ts` |
| UI rendering | `components/<feature>/<Name>.tsx` |
| Page assembly & routing target | `pages/<Name>.tsx` |
| Global state | `stores/` |
| Enums | `utils/enums.ts` |
| Utility functions | `utils/utility.ts` |
| Static values & lookups | `constants/` |
| Static files | `assets/` → re-exported via `assets/index.ts` |

---
name: frontend-architecture
description: >
  Enforce the established frontend architecture when generating, refactoring, or reviewing
  client-side React/TypeScript code. Activate this skill whenever the user asks to create
  a new page, add a component, build a feature, write a hook, set up a store, define an
  API call, or scaffold any frontend module. Also trigger on phrases like "add a page",
  "create a form for...", "build the frontend for...", "add a new component", "wire up a
  new API call", or "scaffold a feature".
---

# Frontend Architecture Skill

This skill defines the **layered architecture**, **hook-based logic separation pattern**,
**folder conventions**, **API layer design**, and **code patterns** used across all
frontend projects. Every piece of generated frontend code **MUST** follow these rules.
No exceptions.

---

## Table of Contents

1. [Technology Stack](#1-technology-stack)
2. [Folder Structure](#2-folder-structure)
3. [Architecture Layers](#3-architecture-layers)
4. [Core Design Principle: UI/Logic Separation via Hooks](#4-core-design-principle-uilogic-separation-via-hooks)
5. [Layer-by-Layer Guide with Examples](#5-layer-by-layer-guide-with-examples)
   - 5.1 [Models Layer (Interfaces)](#51-models-layer-interfaces)
   - 5.2 [API Layer](#52-api-layer)
   - 5.3 [Stores Layer (Zustand)](#53-stores-layer-zustand)
   - 5.4 [Hooks Layer](#54-hooks-layer)
   - 5.5 [Pages Layer](#55-pages-layer)
   - 5.6 [Components Layer](#56-components-layer)
6. [Routing and Route Guards](#6-routing-and-route-guards)
7. [Styling Pattern](#7-styling-pattern)
8. [Error Handling Pattern](#8-error-handling-pattern)
9. [Internationalization (i18n)](#9-internationalization-i18n)
10. [Supporting Infrastructure](#10-supporting-infrastructure)
11. [Wiring a New Feature End-to-End](#11-wiring-a-new-feature-end-to-end)
12. [Checklist for New Features](#12-checklist-for-new-features)

---

## 1. Technology Stack

| Concern              | Technology                                 |
|----------------------|--------------------------------------------|
| Framework            | React 19 with TypeScript                   |
| Build Tool           | Vite 7                                     |
| Routing              | React Router DOM 7                         |
| State Management     | Zustand 5                                  |
| UI Library           | Ant Design (antd) 6                        |
| HTTP Client          | Axios                                      |
| Internationalization | i18next + react-i18next                    |
| Path Aliases         | `@/*` → `src/*` via Vite + tsconfig paths  |
| Icons                | @ant-design/icons + custom SVG components  |
| Auth                 | JWT (access token in Zustand, refresh token in httpOnly cookie) |

---

## 2. Folder Structure

```
src/
├── main.tsx                    # React entry point (renders App)
├── App.tsx                     # Router + route definitions + providers
├── index.css                   # Global CSS overrides
├── i18n.ts                     # i18next initialization
├── api/                        # API layer — one class per domain
│   ├── index.ts                # ★ Barrel — instantiates all API classes
│   ├── authApi.ts
│   ├── accountApi.ts
│   ├── workshopFactionApi.ts
│   ├── fileApi.ts
│   └── ...
├── assets/                     # Static assets
│   ├── index.ts                # Barrel export for all assets
│   ├── icons.tsx               # Custom SVG icon components
│   └── images/                 # Static image files
├── components/                 # Reusable UI components (organized by domain)
│   ├── global/                 # App-wide layout components
│   │   ├── ProtectedRoutes.tsx
│   │   ├── GlobalLogic.tsx
│   │   ├── MainCommonWrap.tsx
│   │   ├── Header.tsx
│   │   ├── PageLoading.tsx
│   │   ├── Wrapper.tsx
│   │   ├── InitModal.tsx
│   │   ├── common/             # Shared UI primitives
│   │   └── form/               # Shared form components
│   ├── faction/                # Domain-specific modal/form components
│   │   ├── CreateFactionModal.tsx
│   │   ├── EditFactionModal.tsx
│   │   └── FactionDisplayModal.tsx
│   ├── workshop/               # Workshop tab content components
│   │   ├── workshopFaction/
│   │   │   ├── WorkshopFaction.tsx
│   │   │   └── WorkshopFactionCard.tsx
│   │   └── ...
│   └── settings/               # Settings tab content components
├── config/
│   ├── axiosConfig.ts          # Axios instances (public + private with interceptors)
│   └── i18n.d.ts               # i18next type augmentation
├── constants/
│   ├── theme.ts                # Ant Design theme token overrides
│   ├── selections.ts           # Static selection data
│   └── locales/
│       └── en.json             # English translations
├── hooks/                      # ★ Logic layer — ALL business logic lives here
│   ├── global/                 # App-wide hooks
│   │   ├── useProtectedRoutes.ts
│   │   ├── useGlobalLogic.ts
│   │   ├── useMainCommonWrap.tsx
│   │   ├── useToken.ts
│   │   ├── useStaticModal.tsx
│   │   ├── useNotification.tsx
│   │   ├── useHeader.tsx
│   │   └── useInitModal.ts
│   ├── login/
│   │   └── useLogin.ts
│   ├── register/
│   │   └── useRegister.ts
│   ├── faction/                # Domain-specific hooks
│   │   ├── useCreateFaction.tsx
│   │   ├── useEditFaction.tsx
│   │   └── useFactionDisplayModal.ts
│   ├── workshop/
│   │   ├── useWorkshop.tsx
│   │   ├── workshopFaction/
│   │   └── ...
│   └── ...
├── models/                     # TypeScript interfaces and types
│   ├── globalInterfaces.ts     # FetchResponse, ErrorResponse
│   ├── authInterfaces.ts
│   ├── accountInterfaces.ts
│   ├── factionInterfaces.ts
│   └── ...
├── stores/                     # Zustand stores (global state)
│   ├── useTokenStore.ts
│   ├── useAccountStore.ts
│   ├── useReferenceStore.ts
│   └── useSidebarStore.ts
└── utils/
    ├── enums.ts                # All const object enums
    ├── utility.ts              # Helper functions (catchFetchError, formatters)
    └── mapMath.ts              # Domain-specific utility
```

### Naming Conventions

| File Type        | Pattern                              | Example                              |
|------------------|--------------------------------------|--------------------------------------|
| Page             | `{PageName}.tsx`                     | `Login.tsx`, `Workshop.tsx`          |
| Component        | `{ComponentName}.tsx`                | `CreateFactionModal.tsx`             |
| Hook             | `use{HookName}.ts` or `.tsx`         | `useLogin.ts`, `useCreateFaction.tsx`|
| API class        | `{domain}Api.ts`                     | `workshopFactionApi.ts`              |
| Model/Interface  | `{domain}Interfaces.ts`              | `factionInterfaces.ts`               |
| Store            | `use{Name}Store.ts`                  | `useTokenStore.ts`                   |
| Enum file        | `enums.ts` (all in one file)         | `utils/enums.ts`                     |

**When to use `.ts` vs `.tsx` for hooks:**
- Use `.ts` when the hook returns only data/functions (no JSX).
- Use `.tsx` when the hook creates JSX elements (e.g., menu items with icon components, notification content).

---

## 3. Architecture Layers

The architecture follows a strict layered design with clear separation:

```
  Page → Hook → API → Backend
         ↕
       Store (Zustand)
         ↕
      Component → Hook → API
```

### Layer Responsibilities

| Layer            | Responsibility                                                      | Contains                     |
|------------------|---------------------------------------------------------------------|------------------------------|
| **Page**         | Route target. Pure UI rendering. Delegates ALL logic to a hook.     | JSX + styling only           |
| **Hook**         | ALL business logic: state, effects, handlers, API calls, navigation | `useState`, `useEffect`, handlers |
| **Component**    | Reusable UI pieces. May have their own hooks for encapsulated logic | JSX + optional hook usage    |
| **API**          | HTTP communication layer. Wraps axios calls per domain.             | Classes with typed methods   |
| **Store**        | Global client-side state (Zustand). Minimal logic.                  | State + setters              |
| **Model**        | TypeScript interfaces/types for API DTOs and return types           | `interface`, `type`          |

### Critical Rules

1. **Pages contain ZERO logic** — no `useState`, no `useEffect`, no handlers. ALL logic goes into the hook.
2. **Hooks are the brain** — they own state, effects, event handlers, API calls, and navigation.
3. **Components CAN have their own hooks** — for encapsulated, reusable logic (e.g., `CreateFactionModal` uses `useCreateFaction`).
4. **API classes are stateless** — they only make HTTP requests and return typed results.
5. **Stores hold only global state** — they have minimal logic (just setters).

---

## 4. Core Design Principle: UI/Logic Separation via Hooks

This is the **most important architectural pattern** in this codebase. Every page and
complex component has a corresponding custom hook that owns ALL of its logic.

### The Pattern

```
┌─────────────────────┐     ┌────────────────────────┐
│     Page (UI)        │     │     Hook (Logic)        │
│                      │     │                         │
│  - JSX only          │◄────│  - useState             │
│  - Destructures      │     │  - useEffect            │
│    hook return       │     │  - Event handlers       │
│  - Inline styles     │     │  - API calls            │
│  - No useState       │     │  - Navigation           │
│  - No useEffect      │     │  - Form management      │
│  - No handlers       │     │  - Loading states       │
└─────────────────────┘     └────────────────────────┘
```

### Why This Matters

- **Testability**: Logic can be tested independently of rendering.
- **Readability**: Pages are purely declarative — you see the UI structure at a glance.
- **Reusability**: Logic hooks can be shared across different UI presentations.
- **Separation of Concerns**: UI changes don't touch logic, logic changes don't touch UI.

---

## 5. Layer-by-Layer Guide with Examples

### 5.1 Models Layer (Interfaces)

Each domain has a `{domain}Interfaces.ts` file containing:
- **DTOs** (`Create___DTO`, `Update___DTO`) for data sent to the API.
- **Return types** (`___Return`) for data received from the API.
- **UI types** (base entity types used by components).

**Rules:**
- Use `interface` for DTOs (input shapes).
- Use `type` for return types (output shapes).
- Use `extends` and intersection (`&`) to compose types.
- Global types (`FetchResponse`, `ErrorResponse`) live in `globalInterfaces.ts`.

**Example: `globalInterfaces.ts`**

```typescript
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
    }
}
```

**Example: `factionInterfaces.ts`**

```typescript
export interface CreateFactionDTO {
    image: string | null;
    name: string;
    description: string;
    color: string;
}

export interface UpdateWorkshopFactionDTO extends CreateFactionDTO {
    workshopFactionId: number;
    isImageUpdated: boolean;
}

export type Faction = {
    image: string | null;
    name: string;
    description: string;
    color: string;
    createdAt: Date;
}

export type WorkshopFactionReturn = Faction & {
    workshopFactionId: number;
    accountId: string;
}
```

---

### 5.2 API Layer

API classes wrap Axios calls with full type safety using the `catchFetchError` utility.
Every method returns a discriminated tuple: `[undefined, T] | [ErrorResponse]`.

**Rules:**
- One class per domain: `{domain}Api.ts`.
- All classes are instantiated in `api/index.ts` (barrel/composition root).
- Use `axiosPublic` for unauthenticated routes, `axiosPrivate` for authenticated routes.
- Type the Axios response generic as `FetchResponse<T>` to extract `data.data`.
- Every method uses `catchFetchError` wrapper for consistent error handling.
- Import types with `import type` syntax.

**Error handling pattern (the `catchFetchError` utility):**

```typescript
// utils/utility.ts
export const catchFetchError = <T>(promise: Promise<T>): Promise<[undefined, T] | [ErrorResponse]> => {
    return promise.then(data => {
        return [undefined, data] as [undefined, T]
    }).catch(err => {
        return [err];
    });
}
```

**Example: `workshopFactionApi.ts`**

```typescript
import { axiosPrivate } from "@/config/axiosConfig";

// utils
import { catchFetchError } from "@/utils/utility";

// interfaces
import type { FetchResponse, ErrorResponse } from "@/models/globalInterfaces";
import type {
    CreateFactionDTO,
    UpdateWorkshopFactionDTO,
    WorkshopFactionReturn
} from "@/models/factionInterfaces";

export class WorkshopFactionApi {
    async getFactions(): Promise<[undefined, WorkshopFactionReturn[]] | [ErrorResponse]> {
        const [error, res] = await catchFetchError(axiosPrivate.get<FetchResponse<WorkshopFactionReturn[]>>(
            '/workshop-faction'
        ));

        if (error) return [error];
        return [error, res.data.data];
    }

    async createFaction(data: CreateFactionDTO): Promise<[undefined, WorkshopFactionReturn] | [ErrorResponse]> {
        const [error, res] = await catchFetchError(axiosPrivate.post<FetchResponse<WorkshopFactionReturn>>(
            '/workshop-faction',
            data,
        ));

        if (error) return [error];
        return [error, res.data.data];
    }

    async deleteFaction(id: number | string): Promise<[undefined, boolean] | [ErrorResponse]> {
        const [error] = await catchFetchError(axiosPrivate.delete<FetchResponse<boolean>>(
            `/workshop-faction/${id}`
        ));

        if (error) return [error];
        return [error, true];
    }
}
```

**Barrel export in `api/index.ts`:**

```typescript
import { WorkshopFactionApi } from "./workshopFactionApi";
import { AuthApi } from "./authApi";
import { FileApi } from "./fileApi";

export const workshopFactionApi = new WorkshopFactionApi();
export const authApi = new AuthApi();
export const fileApi = new FileApi();
```

**Axios Configuration (`config/axiosConfig.ts`):**

Two Axios instances exist:
- **`axiosPublic`** — no auth header, used for login/register/public endpoints.
- **`axiosPrivate`** — automatically attaches `Authorization: Bearer <token>` header via request interceptor, and handles 401 token refresh via response interceptor.

```typescript
import axios from "axios";
import useTokenStore from "@/stores/useTokenStore";

export const axiosPublic = axios.create({
    baseURL: import.meta.env.VITE_API_BASE_URL
});

export const axiosPrivate = axios.create({
    baseURL: import.meta.env.VITE_API_BASE_URL,
    withCredentials: true
});

// Request interceptor: attach access token
axiosPrivate.interceptors.request.use(
    config => {
        const accessToken = useTokenStore.getState().accessToken;
        if (accessToken) {
            config.headers.Authorization = `Bearer ${accessToken}`;
        }
        return config;
    },
    (error) => Promise.reject(error)
);

// Response interceptor: auto-refresh on 401
axiosPrivate.interceptors.response.use(
    (response) => response,
    async (error) => {
        const originalRequest = error.config;

        if (error.response.status !== 401 || originalRequest._retry) {
            return Promise.reject(error);
        }

        originalRequest._retry = true;

        try {
            const res = await axiosPrivate.get('/token');
            const newAccessToken = res.data.data.accessToken ? res.data.data.accessToken : "";

            useTokenStore.getState().setAccessToken(newAccessToken);
            originalRequest.headers["Authorization"] = `Bearer ${newAccessToken}`;

            return axiosPrivate(originalRequest);
        } catch (err) {
            window.location.href = '/login';
            return Promise.reject(err);
        }
    }
)
```

---

### 5.3 Stores Layer (Zustand)

Zustand stores hold **global client-side state**. They are minimal — just state + setters.

**Rules:**
- One store per concern: `use{Name}Store.ts`.
- Define the state interface inside the store file.
- Use `create<StateType>((set) => ({ ... }))`.
- Export as `default`.
- Stores should NOT contain business logic — just state and setters.
- Access store state outside React components using `useStore.getState()`.

**Example: `useTokenStore.ts`**

```typescript
import { create } from "zustand";

interface TokenState {
    accessToken: string;
    setAccessToken: (token: string) => void;
    removeAccessToken: () => void;
}

const useTokenStore = create<TokenState>((set) => ({
    accessToken: "",
    setAccessToken: (token) => set({ accessToken: token }),
    removeAccessToken: () => set({ accessToken: "" }),
}));

export default useTokenStore;
```

**Example: `useAccountStore.ts`**

```typescript
import { create } from "zustand";

// interface
import type { AccountStateDTO } from "@/models/accountInterfaces";

interface AccountState {
    accountId: string,
    username: string,
    email: string,
    setAccount: (account: AccountStateDTO) => void;
    removeAccount: () => void;
    setUsername: (username: string) => void;
}

const useAccountStore = create<AccountState>((set) => ({
    accountId: "",
    username: "",
    email: "",
    setAccount: (account: AccountStateDTO) => set({ ...account }),
    removeAccount: () => set({ accountId: "", username: "", email: "" }),
    setUsername: (username: string) => set({ username }),
}));

export default useAccountStore;
```

---

### 5.4 Hooks Layer

Hooks are the **core of the architecture**. They contain ALL business logic that pages
and components need.

**Rules:**
- Hooks live in `hooks/{domain}/` directories.
- Each page has exactly one primary hook: `use{PageName}.ts`.
- Each complex component may have its own hook: `use{ComponentName}.ts`.
- Hooks return an object with all the state, handlers, and data the UI needs.
- Hooks call API instances from `@/api`.
- Hooks call stores for global state read/write.
- Hooks manage `useState` for local state, `useEffect` for side effects.
- Hooks handle navigation via `useNavigate()`.
- Hooks use Ant Design `Form.useForm()` for form state management.
- Use `useStaticModal()` for error/success/confirmation modals.
- Use `useNotification()` for toast notifications.

**Hook Types:**

| Type              | Purpose                                           | Example                     |
|-------------------|---------------------------------------------------|-----------------------------|
| **Page hook**     | Owns ALL logic for a page                         | `useLogin`, `useRegister`   |
| **Component hook**| Encapsulates logic for a reusable component       | `useCreateFaction`          |
| **Global hook**   | App-wide logic (auth guard, layout, data loading) | `useProtectedRoutes`, `useGlobalLogic` |
| **Utility hook**  | Shared helper logic (modals, notifications)       | `useStaticModal`, `useNotification` |

**Example: Page Hook — `useLogin.ts`**

```typescript
import { useEffect, useState } from "react";
import { Form, type FormProps } from "antd";

// hooks
import { useNavigate } from "react-router-dom";
import useStaticModal from "@/hooks/global/useStaticModal";
import useToken from "@/hooks/global/useToken";

// api
import { authApi } from "@/api";

// interfaces
interface LoginForm {
    identifier: string;
    password: string;
}

export default function useLogin() {

    const navigate = useNavigate();
    const { errorModal, serverErrorModal } = useStaticModal();
    const { isLoggedIn } = useToken();

    const [loginForm] = Form.useForm();

    const [loading, setLoading] = useState(false);
    const [pageLoad, setPageLoad] = useState(true);
    const [errorMsg, setErrorMsg] = useState<string>("");

    useEffect(() => {
        checkLoggedIn();
    }, []);

    const checkLoggedIn = async () => {
        const loggedIn = await isLoggedIn();
        if (loggedIn) navigate('/');
        setPageLoad(false);
    }

    const submitLoginData: FormProps<LoginForm>['onFinish'] = async (values) => {
        setLoading(true);

        try {
            const [err, data] = await authApi.login({
                identifier: values.identifier,
                password: values.password,
            });

            if (err) {
                if (err.status === 400) {
                    const error = err.response.data.schemaErrors ? err.response.data.schemaErrors[0] : undefined;
                    if (!error) return;
                    errorModal(undefined, `${error.field} is ${error.message}`);
                    return;
                }

                if (err.status === 404) {
                    setErrorMsg(err.response.data.userMessage);
                    return;
                }

                serverErrorModal();
                return;
            }

            navigate(`/verification?token=${data.verificationToken}`);

        } finally {
            setLoading(false);
        }
    };

    return {
        loginForm,
        loading,
        errorMsg,
        pageLoad,
        submitLoginData,
    }
}
```

**Example: Component Hook — `useCreateFaction.tsx`**

Component hooks receive callbacks and data from the parent via **function parameters**
(not props — hooks aren't components).

```typescript
import { useState } from "react";
import { Form, type FormProps } from "antd";

// interfaces
import type { CreateFactionDTO } from "@/models/factionInterfaces";

interface CreateFactionFormValues {
    name: string;
    description: string;
    color: string;
}

export default function useCreateFaction(
    onClose: () => void,
    onCreateSubmit: (data: CreateFactionDTO) => Promise<void>
) {

    const [createFactionForm] = Form.useForm<CreateFactionFormValues>();

    const [submitLoad, setSubmitLoad] = useState<boolean>(false);
    const [imageUrl, setImageUrl] = useState<string>("");

    const handleFileChange = (url: string) => {
        setImageUrl(url);
    };

    const submitCreateFaction: FormProps<CreateFactionFormValues>["onFinish"] = async (values) => {
        const submitData: CreateFactionDTO = {
            image: imageUrl,
            name: values.name,
            description: values.description,
            color: values.color,
        };

        setSubmitLoad(true);

        try {
            await onCreateSubmit(submitData);
            handleCloseModal();
        } finally {
            setSubmitLoad(false);
        }
    };

    const handleCloseModal = () => {
        createFactionForm.resetFields();
        setImageUrl("");
        onClose();
    };

    return {
        createFactionForm,
        submitLoad,
        handleFileChange,
        submitCreateFaction,
        handleCloseModal,
    };
}
```

---

### 5.5 Pages Layer

Pages are **pure UI**. They destructure everything from their hook and render JSX.

**Rules:**
- Pages live in `pages/` as flat files.
- Each page exports a `default function`.
- The FIRST thing inside the function is the hook call — destructure all values.
- Pages contain ZERO `useState`, ZERO `useEffect`, ZERO event handlers.
- Pages define their styles using a `styles` object at the bottom of the file.
- Use `useTranslation()` in the page for localized strings.
- Handle loading states with early returns: `if (pageLoad) return <PageLoading />;`.

**Example: `Login.tsx`**

```tsx
import { Button, Input, Form, Alert } from 'antd';

// components
import PageLoading from '@/components/global/PageLoading';

// hooks
import { useTranslation } from 'react-i18next';
import useLogin from '@/hooks/login/useLogin';

export default function Login() {

    const {
        loginForm,
        loading,
        errorMsg,
        pageLoad,
        submitLoginData,
    } = useLogin();

    const { t } = useTranslation();

    if (pageLoad) {
        return <PageLoading />
    }

    return (
        <div style={styles.page}>
            {/* Pure JSX — no logic, no handlers defined here */}
            <Form
                form={loginForm}
                onFinish={submitLoginData}
            >
                {/* Form fields... */}
                <Button
                    type="primary"
                    htmlType="submit"
                    loading={loading}
                >
                    {t("login.enterRealm")}
                </Button>
            </Form>
        </div>
    )
}

const styles: { [key: string]: React.CSSProperties } = {
    page: {
        width: '100%',
        minHeight: '100vh',
        display: 'flex',
        justifyContent: 'center',
        backgroundColor: '#EAE3D2',
    },
    // ... more styles
};
```

---

### 5.6 Components Layer

Components are organized by domain inside `components/`.

**Rules:**
- Global/layout components live in `components/global/`.
- Domain-specific components live in `components/{domain}/`.
- Workshop domain components live in `components/workshop/{workshopDomain}/`.
- Complex components with their own logic use a dedicated hook.
- Simple presentational components receive data via props.
- Components use Ant Design components for UI primitives.

**Component organization mirrors the hooks structure:**

```
components/faction/CreateFactionModal.tsx  ←→  hooks/faction/useCreateFaction.tsx
components/faction/EditFactionModal.tsx    ←→  hooks/faction/useEditFaction.tsx
components/faction/FactionDisplayModal.tsx ←→  hooks/faction/useFactionDisplayModal.ts
```

---

## 6. Routing and Route Guards

### Route Structure in `App.tsx`

```tsx
<ConfigProvider theme={theme}>
    <AntApp>
        <Router>
            <Routes>
                {/* Public routes — no auth required */}
                <Route path="/register" element={<Register />} />
                <Route path="/login" element={<Login />} />
                <Route path="/verification" element={<Verification />} />

                {/* Protected routes — require authentication */}
                <Route element={<ProtectedRoutes />}>
                    <Route element={<GlobalLogic />}>
                        <Route element={<MainCommonWrap />}>
                            <Route path="/" element={<Dashboard />} />
                            <Route path="/workshop" element={<Workshop />} />
                            <Route path="/settings" element={<Settings />} />
                        </Route>
                    </Route>
                </Route>

                {/* 404 */}
                <Route path="*" element={<NotFound />} />
            </Routes>
        </Router>
    </AntApp>
</ConfigProvider>
```

### Route Guard Components (Layout Routes)

Three nested layout routes protect authenticated pages:

1. **`ProtectedRoutes`** — Checks if user is logged in. Redirects to `/login` if not.
   Uses `useProtectedRoutes` hook. Renders `<Outlet />` when authenticated.

2. **`GlobalLogic`** — Fetches account data and reference data on mount. Stores
   results in Zustand stores. Shows `<PageLoading />` until all data is loaded.
   Uses `useGlobalLogic` hook.

3. **`MainCommonWrap`** — Renders the app shell (header + sidebar + content area).
   Uses `useMainCommonWrap` hook. Sidebar navigation uses enum keys.

Each layout route component follows the same pattern:
- Call its hook → get state → conditional render (loading vs content) → `<Outlet />`.

---

## 7. Styling Pattern

### Inline Style Objects

All styling uses **inline React style objects** defined as a `const styles` object at
the bottom of each file. No CSS modules, no styled-components, no Tailwind.

```typescript
const styles: { [key: string]: React.CSSProperties } = {
    page: {
        width: '100%',
        minHeight: '100vh',
        display: 'flex',
        justifyContent: 'center',
        backgroundColor: '#EAE3D2',
    },
    container: {
        maxWidth: '75rem',
        flex: '1',
        padding: '2rem',
    },
    // ...
};
```

**Rules:**
- The type is always `{ [key: string]: React.CSSProperties }`.
- Styles are defined at module scope, below the component function.
- Use the theme colors consistently (see theme tokens below).
- Use `index.css` ONLY for global overrides and Ant Design component overrides.

### Theme Tokens

The Ant Design theme is configured in `constants/theme.ts`:

```typescript
export const theme = {
    token: {
        colorPrimary: '#D95C14',     // Autumn Rust
        colorTextBase: '#3E4A3D',     // Forest Canopy
        colorBgBase: '#F5F1E7',       // Birch Wood
        colorBorder: '#C2BAA6',       // Dried Twig
        fontFamily: "'Palatino Linotype', 'Book Antiqua', Palatino, serif",
        borderRadius: 8,
    },
    components: { /* component-level overrides */ }
};
```

Reference these tokens via `theme.token.colorPrimary` etc. in hooks and components
that build custom UI (modals, notifications).

---

## 8. Error Handling Pattern

### In Hooks (API Call Error Handling)

```typescript
const [err, data] = await someApi.someMethod(input);

if (err) {
    // Schema validation error (400)
    if (err.status === 400) {
        const error = err.response.data.schemaErrors?.[0];
        if (!error) return;
        errorModal(undefined, `${error.field} is ${error.message}`);
        return;
    }

    // Not found (404)
    if (err.status === 404) {
        setErrorMsg(err.response.data.userMessage);
        return;
    }

    // Conflict (409)
    if (err.status === 409) {
        setErrorMsg(err.response.data.userMessage);
        return;
    }

    // Catch-all: server error
    serverErrorModal();
    return;
}

// Success path — use `data`
```

### Modal and Notification Hooks

- **`useStaticModal()`** — Returns `successModal`, `errorModal`, `warningModal`,
  `infoModal`, `serverErrorModal`, `confirmationModal`.
- **`useNotification()`** — Returns `successNotification`, `errorNotification`,
  `warningNotification`, `infoNotification`.

Use modals for blocking errors/confirmations. Use notifications for non-blocking feedback.

---

## 9. Internationalization (i18n)

### Setup

- i18next is initialized in `src/i18n.ts`.
- Translations live in `constants/locales/en.json`.
- Type safety is provided via `config/i18n.d.ts`.

### Usage in Components/Pages

```typescript
import { useTranslation } from 'react-i18next';

const { t } = useTranslation();

// Use in JSX
<Title>{t('login.title')}</Title>
<Text>{t('global.fieldRequired')}</Text>
```

### Usage in Hooks

Hooks may also use `useTranslation()` when constructing menu items or
other data structures that include translated labels.

**Rule:** All user-facing strings MUST use the `t()` function. Never hardcode
display text directly.

---

## 10. Supporting Infrastructure

### Enums (`utils/enums.ts`)

Enums use `const` object pattern with `as const` for type safety:

```typescript
export const WorkshopMenuEnum = {
    WORLDS: "worlds",
    CHARACTERS: "characters",
    RACES: "races",
    CLASSES: "classes",
    FACTIONS: "factions",
    MONSTERS: "monsters",
    ITEMS: "items",
    SPELLS: "spells",
    FEATS: "feats",
} as const;

export const SidebarMenuEnum = {
    DASHBOARD: "dashboard",
    CAMPAIGNS: "campaigns",
    WORKSHOP: "workshop",
} as const;
```

**Rules:**
- All enums live in `utils/enums.ts`.
- Use `as const` pattern (not TypeScript `enum` keyword).
- Never compare against raw strings in logic — use enum values.

### Assets (`assets/`)

- Custom SVG icons are React components in `assets/icons.tsx`.
- Image files live in `assets/images/`.
- Everything is re-exported from `assets/index.ts`.

---

## 11. Wiring a New Feature End-to-End

When adding a new domain feature (e.g., "Workshop Skill"), follow this exact order:

### Step 1: Models — `models/skillInterfaces.ts`
Define `CreateSkillDTO`, `UpdateSkillDTO`, `Skill`, and `WorkshopSkillReturn`.

### Step 2: API — `api/workshopSkillApi.ts`
Create the API class with CRUD methods. Register in `api/index.ts`:
```typescript
export const workshopSkillApi = new WorkshopSkillApi();
```

### Step 3: Hook (Workshop Tab) — `hooks/workshop/workshopSkill/useWorkshopSkill.tsx`
Create the hook that manages the list, CRUD operations, and modal states.

### Step 4: Hook (Create) — `hooks/skill/useCreateSkill.tsx`
Create the hook for the create modal form logic.

### Step 5: Hook (Edit) — `hooks/skill/useEditSkill.tsx`
Create the hook for the edit modal form logic.

### Step 6: Component (Tab Content) — `components/workshop/workshopSkill/WorkshopSkill.tsx`
Build the workshop tab content (card grid, modals).

### Step 7: Component (Card) — `components/workshop/workshopSkill/WorkshopSkillCard.tsx`
Build the card component for the grid.

### Step 8: Component (Modals) — `components/skill/CreateSkillModal.tsx`, `EditSkillModal.tsx`
Build the modal form components, each using their respective hooks.

### Step 9: Register in Workshop
Add the new tab to `WorkshopMenuEnum` in `utils/enums.ts`, add the menu item in
`useWorkshop.tsx`, and add the conditional render in `Workshop.tsx`.

### Step 10: Translations
Add all new strings to `constants/locales/en.json`.

---

## 12. Checklist for New Features

Before considering a frontend feature complete, verify:

- [ ] **Interfaces** defined in `models/{domain}Interfaces.ts` with DTOs and return types
- [ ] **API class** created and registered in `api/index.ts`
- [ ] **Hook(s)** created in `hooks/{domain}/` — page hooks, component hooks as needed
- [ ] **Page** (if new route) created in `pages/` — pure UI, zero logic
- [ ] **Component(s)** created in `components/{domain}/` — each with its own hook if complex
- [ ] **UI/Logic separation** enforced — no `useState`/`useEffect`/handlers in pages
- [ ] **Translations** added to `constants/locales/en.json` — no hardcoded display text
- [ ] **Enums** used for all string comparisons — added to `utils/enums.ts`
- [ ] **Error handling** follows the pattern: check `err.status`, use `errorModal`/`serverErrorModal`
- [ ] **Loading states** managed with `useState<boolean>` + `try/finally` pattern
- [ ] **Styles** defined as inline style objects with `{ [key: string]: React.CSSProperties }`
- [ ] **Theme colors** used consistently from `constants/theme.ts`
- [ ] **API tuple pattern** used: `const [err, data] = await api.method()`
- [ ] **Route registered** in `App.tsx` (public or inside `ProtectedRoutes` nesting)
- [ ] **Workshop tab** registered in `useWorkshop.tsx` + `Workshop.tsx` (if workshop feature)

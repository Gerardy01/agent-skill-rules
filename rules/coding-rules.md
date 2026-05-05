Enforce a strict set of code quality and style rules when writing, reviewing, or refactoring code. Use this skill whenever the user asks to write new code, refactor existing code, review a file, fix a bug, add a feature, or generate any TypeScript / JavaScript. Also trigger on phrases like "clean this up", "refactor this", "write a function for...", "add a method", or "review my code". These rules are always active — apply them automatically without waiting to be asked.

# Coding Rules

These rules apply to **all code written or modified**. They are non-negotiable defaults.
The only exceptions are noted per rule. When in doubt, follow the rule.

---

## Rule 1 — No `any` Type

**Never use `any`** unless it is genuinely the only option (e.g., interfacing with an
untyped third-party library where no `@types` package exists).

- Before using `any`, try: `unknown`, a union type, a generic `<T>`, or a properly
  typed interface.
- If `any` seems necessary, **stop and ask the user** before proceeding:
  > "I'd need to use `any` here for `[reason]`. Would you like me to proceed, or should
  > we define a proper type together?"
- Never silently cast with `as any` to suppress an error.

```ts
// ❌ Never
function parse(data: any): any { ... }

// ✅ Preferred
function parse<T>(data: unknown): T { ... }
```

---

## Rule 2 — No Nested Conditionals — Use Guard Clauses

**Avoid nested `if` blocks.** Return (or throw) early to handle edge cases and keep
the happy path un-indented.

- If you find yourself writing `if (...) { if (...) { ... } }`, restructure with guards.
- Exception: a single nested `if` inside a loop body is acceptable if restructuring
  would hurt readability more than it helps.

```ts
// ❌ Nested — hard to follow
function process(order: Order) {
  if (order) {
    if (order.isPaid) {
      if (order.items.length > 0) {
        ship(order);
      }
    }
  }
}

// ✅ Guard clauses — intent is clear immediately
function process(order: Order) {
  if (!order) return;
  if (!order.isPaid) return;
  if (order.items.length === 0) return;

  ship(order);
}
```

---

## Rule 3 — Explicit Return Types on All Functions

Every function and method **must declare its return type explicitly**.

- Applies to: regular functions, arrow functions, class methods, async functions.
- `void` and `Promise<void>` must also be written out explicitly.
- Do not rely on TypeScript inference for public-facing or exported functions.

```ts
// ❌ Implicit return type
const getUser = (id: string) => fetchUser(id);

// ✅ Explicit return type
const getUser = (id: string): Promise<User> => fetchUser(id);

// ✅ Async with void
async function syncCache(): Promise<void> {
  await cache.refresh();
}
```

---

## Rule 4 — `const` First, `let` Only When Necessary, Never `var`

- Default to `const` for every declaration.
- Use `let` **only** when the variable must be reassigned after declaration.
- **Never** use `var` — not in any context, not for legacy compatibility.

```ts
// ❌ Never
var counter = 0;

// ❌ Unnecessary let
let name = 'Alice';

// ✅
const name = 'Alice';

// ✅ let is justified here
let total = 0;
for (const item of items) {
  total += item.price;
}
```

---

## Rule 5 — Use Enums for Static String Values in Logic

**Never compare against or pass raw string literals in business logic.** Define an enum
and use it instead.

- Applies to: `if` conditions, `switch` cases, function arguments that accept a fixed
  set of string values, object keys used as identifiers.
- Exception: strings that are user-facing content (labels, messages, translations) do
  not need enums.
- Place enums in a dedicated file (e.g., `enums.ts` or co-located with the domain
  model) rather than inline in a function.

```ts
// ❌ Magic string
if (status === 'active') { ... }

// ✅ Enum
enum UserStatus {
  Active = 'active',
  Inactive = 'inactive',
  Suspended = 'suspended',
}

if (status === UserStatus.Active) { ... }
```

---

## Rule 6 — Descriptive, Unabbreviated Names

Names must communicate intent clearly without requiring context from the reader.

- **No single-letter variables** — exception: loop counters (`i`, `j`, `k`) in simple
  `for` loops and conventional generic type parameters (`T`, `K`, `V`).
- **No abbreviations** — write the full word.
- **No vague names** — `data`, `info`, `temp`, `result`, `obj` are not acceptable
  unless wrapped in a more specific context name.

```ts
// ❌ Abbreviations and vague names
const usr = await getUsr(req.params.id);
const btn = document.querySelector('#btn');
let res: any;

// ✅ Clear, full names
const user = await getUserById(request.params.id);
const submitButton = document.querySelector('#submit-button');
let validationResult: ValidationResult;
```

---

## Rule 7 — Single Responsibility Per Function

**One function does exactly one thing.** If you need the word "and" to describe what
a function does, it must be split into two functions.

- Functions should be short — aim for under 20 lines. If longer, look for extraction
  opportunities.
- Side effects (logging, writing to DB, emitting events) should be isolated into their
  own functions, not mixed with pure logic.
- Compose smaller functions to build larger behaviors.

```ts
// ❌ Does too many things
async function processAndNotifyUser(userId: string): Promise<void> {
  const user = await db.users.findById(userId);
  user.status = UserStatus.Active;
  await db.users.save(user);
  await emailService.sendWelcome(user.email);
  await analyticsService.track('user_activated', { userId });
}

// ✅ Each function has one job
async function activateUser(userId: string): Promise<User> {
  const user = await db.users.findById(userId);
  user.status = UserStatus.Active;
  return db.users.save(user);
}

async function notifyUserActivated(user: User): Promise<void> {
  await emailService.sendWelcome(user.email);
  await analyticsService.track('user_activated', { userId: user.id });
}
```

---

## Rule 8 — No Unused Variables or Imports

**Remove any variable, parameter, or import that is not used.** Do not leave dead code
commented out — delete it. Version control preserves history.

- If a function parameter must be present (e.g., to satisfy a callback signature) but
  isn't used, prefix it with `_` to signal intent:
  `(event, _context) => { ... }`
- Barrel files (`index.ts`) that re-export everything are exempt — unused re-exports
  there are intentional.
- After any refactor, scan for and remove newly orphaned imports.

```ts
// ❌ Unused import and variable
import { formatDate } from './utils'; // never used
import { UserService } from './user.service';

async function getProfile(userId: string): Promise<Profile> {
  const timestamp = Date.now(); // never used
  return UserService.getProfile(userId);
}

// ✅ Clean
import { UserService } from './user.service';

async function getProfile(userId: string): Promise<Profile> {
  return UserService.getProfile(userId);
}
```

---

## Applying the Rules in Practice

### When writing new code
Apply all rules from scratch. Do not produce a first draft and then "fix it up" — write
correctly from the start.

### When refactoring existing code
- Apply all rules to every line you touch.
- If you spot a violation in code you are **not** modifying, flag it with a comment to
  the user rather than silently rewriting untouched code.
  > "I noticed `usr` on line 42 violates the naming rule — want me to rename it while
  > I'm in this file?"

### When reviewing code
Go through the checklist below and report violations with the line, rule number, and a
suggested fix.

---

## Review Checklist

Before submitting any code, verify:

- [ ] **R1** — No `any` used; if present, user was consulted
- [ ] **R2** — No nested `if` blocks; guard clauses used instead
- [ ] **R3** — Every function has an explicit return type
- [ ] **R4** — Only `const` and `let`; zero `var`
- [ ] **R5** — No magic strings in logic; enums defined and used
- [ ] **R6** — All names are descriptive and unabbreviated
- [ ] **R7** — Each function has a single, clearly describable responsibility
- [ ] **R8** — No unused variables, parameters, or imports

# TypeScript Style Guide

Read [general_principles.md](general_principles.md) first. This guide adds the TypeScript specifics.

Assumes TypeScript 4.9 or later (`satisfies`) and an ES2023 target for the non-mutating array methods. Test examples use the Jest or Vitest API (`it.each`, fake timers).

## 2.1 Types

- Use `type` for object shapes, unions and aliases. Use `interface` only when you need declaration merging or `implements` on a class. *(opinion)*
- Name prop or parameter object types clearly. A consistent prefix or suffix (`TProps`, `Props`) is fine when the project uses one.
- Never use `any`. Use `unknown` at untrusted boundaries and narrow it.
- Avoid type assertions (`as`). `as unknown as X` almost always means the model is wrong. Fix the type at its source or narrow with a type guard.
- Do not use non-null assertions (`!`). Handle the missing value or restructure so it cannot be missing.
- Derive types instead of redeclaring them. Reshape existing types with `Pick`, `Omit`, `Partial`, `ReturnType`, `Parameters`, indexed access and `typeof`.
- Let inference work. Annotate exported function return types only when inference is wrong, unreadable or you want a stable public contract. Always annotate parameters. *(opinion)*
- Turn on `strict`. Do not weaken it per file.

```ts
type User = {id: string; name: string; email: string; role: 'admin' | 'member'};

type UserSummary = Pick<User, 'id' | 'name'>;
type NewUser = Omit<User, 'id'>;
type UserPatch = Partial<NewUser>;
```

## 2.2 Modeling

- Model variants as discriminated unions with a literal tag, and narrow with the tag.
- Make illegal states unrepresentable. Replace several booleans with one union.

```ts
// Bad: 8 combinations, most nonsensical
type Request = {isLoading: boolean; isError: boolean; data?: Data; error?: Error};

// Good: 4 states, each with exactly the fields it can have
type Request =
  | {status: 'idle'}
  | {status: 'loading'}
  | {status: 'success'; data: Data}
  | {status: 'error'; error: Error};
```

- Write type guards for variants you branch on often, and reuse them:

```ts
const isCircle = (shape: Shape): shape is Circle => shape.kind === 'circle';
```

- Use `Record<Union, T>` for lookup tables so the compiler forces every case. Use `satisfies` to check a literal against a type while keeping its narrow type.
- Ensure exhaustiveness in `switch` over a union with a `never` check in the default branch.

```ts
const assertNever = (value: never): never => {
  throw new Error(`Unhandled case: ${JSON.stringify(value)}`);
};
```

- Prefer string literal unions for closed sets used in one module. Use an `enum` (or an `as const` object) for a closed set shared across modules that needs a runtime value. Avoid numeric enums. *(opinion)*
- Use `readonly` for data that should not change, and `as const` for literal tables.
- Brand primitives that must not be mixed up (`UserId` vs `OrderId`) when the bug class is real, not by default.

## 2.3 Nulls and optional data

- Choose one representation of "missing" per boundary.
- Prefer `undefined` for optional and avoid `null` unless an external API requires it. *(opinion)*
- Normalize optional data once, at the edge, and pass complete values inside.
- Use optional chaining and nullish coalescing (`?.`, `??`) instead of `&&` chains. Use `??` rather than `||` when `0`, `''` or `false` are valid values.
- Do not sprinkle `?.` everywhere as a defensive habit. If a value is never missing, type it so.

```ts
const items = response.items ?? [];
const label = user.nickname ?? user.name;
```

## 2.4 Functions and modules

- Arrow functions assigned to `const` for functions. *(opinion)*
- Named exports for everything. Avoid `export default`; it makes renames and search harder. *(opinion)*
- Use an options object once you have more than two or three parameters, and destructure it in the signature.
- Destructure with defaults instead of reading and re-checking optional fields inside the body.
- One module, one responsibility. Export only what other modules use.
- Import types with `import type` when the project enforces it. Avoid circular imports; if you hit one, move the shared piece down a layer instead of adding a workaround.
- Avoid barrel files that re-export everything from a deep tree. A small barrel that defines a feature's public surface is fine.

```ts
type CreateInvoiceOptions = {customerId: string; lines: InvoiceLine[]; dueInDays?: number};

export const createInvoice = ({customerId, lines, dueInDays = 30}: CreateInvoiceOptions) => { ... };
```

## 2.5 Collections and data transformation

- Prefer `map`, `filter`, `reduce`, `flatMap`, `some`, `every`, `find` over manual loops for transformations. Use a plain loop when it is clearer or you need early exit with side effects.
- Chain small steps with intermediate names when a chain gets long.
- Do not mutate inputs. Use spread and non-mutating array methods (`toSorted`, `toSpliced`, `with`) or copy first.
- Use `Map` and `Set` for keyed lookups and uniqueness instead of scanning arrays repeatedly.
- Use a utility library such as Lodash for what it does well (`compact`, `groupBy`, `isEqual`, `omit`) instead of rewriting it.

## 2.6 Async code

- `async`/`await` over promise chains.
- Every promise is awaited, returned or explicitly handled. No floating promises.
- Run independent work concurrently with `Promise.all`. Do not `await` in a loop when the iterations do not depend on each other.
- Support cancellation (`AbortController`) for work that can become stale.
- Handle errors where you can act on them. Do not wrap every `await` in `try/catch`.
- Return typed results for expected failures instead of throwing:

```ts
type Result<T> = {ok: true; value: T} | {ok: false; error: string};
```

## 2.7 Validation at the boundary

- Data from outside the program (HTTP, storage, user input, `JSON.parse`) is `unknown` until validated. Validate with a schema library or a hand-written guard, then use the typed value.
- Validators are pure functions that return the error message or nothing, and compose:

```ts
const getNameError = (name?: string) => (name?.trim() ? undefined : 'Name is required');

const getFormErrors = (form: Form) => [getNameError(form.name), getEmailError(form.email)].filter(Boolean);
```

- The UI decides how to show the message. Validators never touch the UI.

## 2.8 Style

- Formatting is the formatter's job. Configure Prettier (or equivalent) and never format by hand.
- Import order is the linter's job. Do not sort by hand.
- Use `const` by default. Use `let` only when reassigning, and prefer restructuring to avoid it. Never `var`.
- Prefer template literals to string concatenation.
- Use strict equality (`===`).
- No `console.log` left in committed code. Keep `console.error` only where it is the intended reporting path.
- Prefer regex literals to `new RegExp(string)`. If a pattern must be dynamic, keep it simple and test it against the production build, since minifiers can mangle escape sequences in strings.

## 2.9 Testing TypeScript

- Pure functions get table-driven tests:

```ts
it.each([
  [0, '0 minutes'],
  [3600, '1 hours'],
  [86400, '1 days'],
])('formats %i seconds as %s', (seconds, expected) => {
  expect(formatDuration(seconds)).toBe(expected);
});
```

- Type-level guarantees (exhaustiveness, `Record` coverage) are tested by the compiler. Run the type checker in CI.
- Fake time with the test framework's timers, not real waits.
- Use small factory functions for test data with sensible defaults and per-test overrides:

```ts
const buildUser = (overrides: Partial<User> = {}): User => ({id: 'u1', name: 'Ada', email: 'ada@example.com', role: 'member', ...overrides});
```

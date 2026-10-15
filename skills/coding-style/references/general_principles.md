# General Principles

Language and framework agnostic. The TypeScript and React guides build on this one. The rules in one screen, and how this guide yields to project conventions, are in [SKILL.md](../SKILL.md).

Code examples are in TypeScript, but the rules apply to any language.

## 1.1 Separation of concerns

Every unit of code should have one reason to change. Ask of any file, function or module: "what is the one thing this does?" If the honest answer contains "and", split it.

Think in four layers, whatever the framework calls them:

| Layer | Responsibility | May depend on |
| --- | --- | --- |
| Entry points | Routes, handlers, `main`. Wire things together, hold no logic. | everything below |
| Presentation | Turn state into output (UI, responses, CLI text). | orchestration, logic, constants |
| Orchestration | Sequence work: fetch, save, confirm, notify, navigate. | logic, constants |
| Pure logic and constants | Compute, validate, transform, describe. No I/O, no framework. | nothing above |

Rules:

- Dependencies point downward. If a lower layer needs something from a higher one, move the shared piece down.
- Pure logic never imports the UI, the network client or the database. This is what makes it trivial to test and reuse.
- An entry point is a few lines: compose and delegate.
- Business rules live in one named place. If the same rule appears in the UI and on the server, it is a shared function, not two copies.
- Side effects (network, disk, time, randomness, global state) sit at the edges and are passed in or wrapped, so the core stays deterministic.

### Structure of a codebase

- Group by feature or domain first, by technical kind second. A reader looking for "invoice" code should find one place.
- One concept per file, and the file is named after what it contains.
- Keep folder depth modest. Nest only when a folder would otherwise hold more than about ten unrelated files.
- Expose a feature through one small public entry (an index or module surface). Other code imports from there and never reaches into internals.
- Tests live next to the code they cover, or in a mirror structure. Pick one and stay consistent.
- Be consistent with the project's file naming convention (kebab, snake or camel). Consistency matters more than which one.

## 1.2 Naming

Names are the primary documentation. Every name should say what a value holds or what a function does, so a reader never has to open the code behind it to find out.

- Name by meaning in the domain, not by how it is built or where it is used. `wasOrderEverShipped`, not `checkFlag2`.
- A long, exact name beats a short, vague one. `usePreventDuplicateSubmissions` needs no comment.
- Functions are verbs: `findUser`, `countOrders`, `sumRefunds`. Not `user`, `orderCount` or `refunds`.
- Variables, constants, classes and types are nouns: `user`, `orderCount`, `MAX_LOGIN_ATTEMPTS`, `InvoiceRepository`.
- Booleans read as questions, usually with a prefix: `isX`, `hasX`, `wasX`, `shouldX`, `canX`.
- Dates and times usually end in `At`: `createdAt`, `expiresAt`, `lastLoginAt`.
- `get` returns a value, `is`/`has` returns a boolean, `to`/`format` converts, `create` builds, `update` and `delete` change state.
- Name the positive, not the negative. `isEnabled`, not `isNotDisabled`.
- No vague words as the whole name: `data`, `info`, `item`, `helper`, `manager`, `utils`, `handle`, `process`, `do`.
- No abbreviations except universal ones (`id`, `url`, `html`). Loop indices (`i`) are fine in a three-line loop.
- A name with "and" or "or" in it is two things. Split it.
- Name intermediate values. Pull a condition into a named constant instead of explaining it:

```ts
const canRefund = order.isPaid && !order.isRefunded && daysSince(order.paidAt) <= REFUND_WINDOW_DAYS;
if (canRefund) refund(order);
```

- Name constants for their meaning, not their value: `MAX_LOGIN_ATTEMPTS`, not `THREE`.
- Consistency: one concept has one name across the codebase. Do not mix `remove`, `delete` and `destroy` for the same idea.

## 1.3 Comments

Default: no comments. The code, names and types are the documentation.

A comment is allowed only when it answers a question the code cannot:

- why a non-obvious workaround exists (link the upstream issue),
- a constraint or invariant that is invisible in the code,
- the contract of a shared function whose name is not enough on its own.

When you write one:

- One line, at most two. Never a paragraph.
- Explain why, not what.
- Doc comments only on shared or public APIs, one sentence, no restating of types.

Never:

- Narrating comments (`// increment the counter`).
- Banner or section comments inside a function. If it needs sections, it is several functions.
- Commented-out code. Version control remembers it.
- A `TODO` without an owner and a ticket.
- Comments about how the file was produced, or which tool wrote it.
- Suppressing a linter or compiler rule without first trying to satisfy it. If you must suppress, add a one-line reason.
- Comments that will silently rot, such as descriptions of callers or of other files.

The test: if you delete the comment and rename something, is the code still clear? Then rename.

## 1.4 Simplicity and size

The goal is the smallest amount of code that is obviously correct.

- A function does one thing and fits on a screen. Split by concern, not by line count.
- Aim for functions under about 30 lines and files under about 200. *(opinion)*
- Flat beats nested. Use guard clauses and early returns instead of pyramids of `if`.
- Keep parameter lists short. More than three parameters usually means a missing concept: group them into a named object.
- No boolean flag parameters that switch behavior (`save(order, true)`). Split into two functions or name the option.
- Do not write code for cases that cannot happen. Trust internal callers and validate only at system boundaries (user input, network, files, other services).
- Do not add options, hooks, flags or configuration nobody uses yet.
- Do not abstract on the second occurrence. On the third, do it, and name the abstraction after the concept it represents, not its shape.
- Duplication is cheaper than the wrong abstraction. Do not merge code that looks alike but changes for different reasons.
- Delete dead code, unused exports and unused parameters in the same change that makes them dead.
- Prefer the standard library and well-known utilities over hand-rolled versions.
- A linter complexity warning is design feedback. Split the function. Do not silence it.
- When you touch a file, leave it smaller or the same size unless you are adding real behavior.

### Prefer data to branching

Repeated conditional chains are a lookup table waiting to be written.

```ts
// Branching
if (status === 'paid') return 'Paid';
if (status === 'pending') return 'Pending';
if (status === 'failed') return 'Failed';
return 'Unknown';

// Data
const STATUS_LABELS: Record<Status, string> = {paid: 'Paid', pending: 'Pending', failed: 'Failed', unknown: 'Unknown'};
```

Use the same idea at larger scale with a registry: define each variant as a small object that implements one interface, and list them in one place. Adding a variant means adding one file and one line, and no `switch` elsewhere changes.

### Build for the system you have (YAGNI)

"You aren't gonna need it." Code for a case that never happens is not free robustness. It is read, tested, changed and debugged like any other code, and every extra path is one more place for state to drift. Enough of it makes a system less stable, not more.

- Before handling a case, ask two questions: can it happen in this system as it is deployed, and what does it cost if it does?
- Answer with facts about the real system, not with what could happen in general: how many instances run, whether there is live traffic, what is migrated when, which data is always present, what is being retired.
- If the case cannot happen, do not handle it. If it can but is rare, let it fail loudly: log it, retry it or surface it, and fix the cause when it shows up.
- Keep a safety net only when it is nearly free and protects a promise the system makes, such as a guard that stops a write from overwriting history.
- Do not add queues, retry schedules, reconciliation jobs, extra columns or extra states for a hypothetical. Each one brings its own invariants to keep true.
- Do not design around something that is going away. Key new code on the identifiers and systems that will still exist.
- When in doubt, ship the simple version. Adding handling once a case is real is cheaper than carrying code that reality never tested.

## 1.5 Functions and data

- Prefer pure functions: same input, same output, no hidden reads or writes.
- Prefer immutable data. Return a modified copy instead of mutating an argument.
- Return early for the unhappy path, then write the main path un-nested.
- One level of abstraction per function. A function either orchestrates named steps or performs low-level work, not both.
- Make illegal states unrepresentable. Model variants explicitly (a tagged union or enum) instead of several booleans that can contradict each other.
- Normalize messy inputs once, at the boundary. Inside, work with clean, complete values.
- Return a value, not a status code plus an out-parameter. Return a small result object when a call has several outcomes.

## 1.6 Error handling

- Handle errors where you can do something useful about them. Otherwise let them propagate.
- Do not swallow errors. Every `catch` either recovers, translates or reports.
- Expected outcomes (validation failure, user cancelled, not found) are return values, not exceptions.
- Exceptions are for the unexpected. Throw and catch typed or clearly named errors, not bare strings.
- Validate at boundaries and fail with a message that tells the reader what to fix.
- User-facing messages say what happened and what to do next. Log details for developers, not for users.
- A failing operation cleans up after itself (`finally`, dispose, rollback).
- Consistent message shape: `Failed to <verb> <thing>` for errors, `<Thing> <past-tense verb>` for success. *(opinion)*

## 1.7 Side effects and state

- Keep state as small as possible. If a value can be derived, derive it. Do not store what you can compute.
- One source of truth per fact. Copies drift.
- Keep state as close as possible to where it is used. Lift it only when two things need it.
- Global mutable state is a last resort. When you need shared state, expose it through a small, typed accessor.
- Every subscription, listener and timer has a matching cleanup.
- Make time and randomness injectable so logic stays testable.

## 1.8 Testing

Tests describe what a user or a caller can observe. They must keep passing when the implementation is refactored and fail when behavior breaks.

### Test behavior, not implementation

- Drive the code through its public surface: the UI a user touches, the function a caller invokes, the endpoint a client hits.
- Assert on outcomes: returned values, rendered output, persisted state, messages shown.
- Do not assert on private functions, internal state, call order, component internals or CSS classes.
- A refactor that keeps behavior the same must not break any test. If it does, the test was coupled to implementation.

### Choose the right level

- **Pure logic:** direct input and output tests, table-driven when there are many cases. Cheapest and most precise. Write these first for anything with rules.
- **Features:** exercise the real feature with its real collaborators, and fake only the outside world (network, clock, storage). This is where most tests belong.
- **End to end:** a handful of critical journeys, no more.
- Fewer, higher-level tests that cover behavior beat many low-level tests that mirror the code.

### Mocking

- Mock only at the system boundary: network, database, filesystem, clock, third-party services.
- Do not mock your own modules to make a test easier. If it is hard to test without mocking your own code, the design needs work.
- Prefer a fake (a small working stand-in with real behavior, like an in-memory store or a mock server that keeps state) over stubs that return canned values.
- Assert that a boundary call happened only when it is the observable outcome and the user cannot see it otherwise (a redirect, an outgoing request).
- Put the system into the state you need through real actions or realistic fake data, not by overriding return values inside the code under test.

### Structure and naming

- Name tests as a sentence about behavior: `allows applying a discount code`, `rejects an expired card`, `shows an empty state when there are no orders`.
- One behavior per test. A test may take several steps to reach a behavior, but it asserts one outcome.
- Arrange, act, assert, in that order, visible without scrolling.
- Repeated setup becomes a helper named after what the user does (`placeOrder`, `signInAs`), not after mechanics.
- Cover the unhappy paths that matter: empty, error, invalid input, unauthorized, boundary values.
- Every bug fix ships with a test that fails without the fix.
- Tests are deterministic. No real time, real network, shared state between tests, or arbitrary sleeps. Wait for conditions, not durations.
- No disabled, skipped or commented-out tests. Fix or delete.
- Tests follow the same rules as production code: clear names, no duplication, no long comments.

## 1.9 Working in an existing codebase

- Follow the conventions of the project you are in. Where they conflict with this guide, the project wins.
- Do not copy a pattern just because it exists nearby. One file is not a convention, and existing code may be legacy. Judge it against the principles here.
- Keep changes focused. Do not mix refactors with behavior changes in a way that makes review hard.
- When you touch a messy area, leave it a little better: rename a confusing thing, delete a dead branch, add the missing test. Do not attempt a rewrite you were not asked for.
- Before adding a helper, search for one that already exists.
- Do not add a dependency for something small. Every dependency is code you own without controlling.
- A review suggestion, human or automated, is not a requirement. Check that its scenario can happen in this system before acting on it. If it cannot, reply with the reason instead of adding code.

## 1.10 Review checklist

- [ ] Each new file and function has one job, and its name says it.
- [ ] Dependencies point downward. Pure logic has no I/O.
- [ ] No comment explains what the code does. No commented-out code.
- [ ] No duplicated branching that could be a lookup table or shared function.
- [ ] No code, option or abstraction that nothing uses yet.
- [ ] No handling for cases that cannot happen in the deployed system. Rare cases fail loudly instead of growing new paths.
- [ ] Errors are handled meaningfully, never swallowed.
- [ ] Tests assert observable behavior and mock only the boundary.
- [ ] Unhappy paths are covered.
- [ ] Formatter, linter and type checker are clean, and no new suppressions were added.

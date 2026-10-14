# React Style Guide

Read [general_principles.md](general_principles.md) first, and [typescript.md](typescript.md) for TSX code. This guide adds the React specifics.

Examples assume these libraries. The principles hold without them, but the API names will differ.

| Area | Library | Where it shows |
| --- | --- | --- |
| Server state | TanStack Query v5 | `mutateAsync`, `isPending`, cache invalidation (3.3, 3.5) |
| Component tests | React Testing Library, `@testing-library/user-event`, `@testing-library/jest-dom` | queries, `user.click`, `toBeInTheDocument` (3.11) |
| Network fakes | Mock Service Worker (MSW) v2 | `server.use(http.put(...))`, `HttpResponse` (3.11) |

In the test examples, `render` is a project helper that wraps Testing Library's `render` with the app's providers and returns a `user-event` instance.

## 3.1 Components

- A function component with a props type.
- Named export. *(opinion)*
- Destructure props in the signature. Give optional props defaults there.
- A component reads a few hooks, derives a few values, and returns JSX. If it does more, extract.
- Business rules go to plain functions. Fetch and save sequences go to hooks. Components render.
- Split by concern, not by size. A screen with viewing, editing and wiring concerns is three components (file names follow the project's convention):

```
price_tag/
  price_tag.tsx          wires data and layout
  price_tag_form.tsx     editable draft state and save
  price_tag_summary.tsx  read-only rendering
```

- A small private component used by one file may live in that file. The moment a second file needs it, give it its own file.
- Keep components pure with respect to their props: same props and state, same output.
- Pages and route components compose providers and a view. They contain no logic.

```tsx
type Props = {order: Order; onClose: () => void};

export const OrderSummary = ({order, onClose}: Props) => {
  if (!order.lines.length) return null;
  return <Summary total={getOrderTotal(order)} onClose={onClose} />;
};
```

## 3.2 Rendering

- Guard with an early return, then render the main case.
- `&&` for one branch, a ternary with `null` for two. Never nest ternaries. Extract a component or a variable instead.
- Name conditions that explain the rendering:

```tsx
const showEmptyState = !isLoading && orders.length === 0;
```

- Always give list items a stable `key` from the data (an id), never the array index for lists that reorder or change.
- Do not define components inside other components. It resets their state on every render.
- Extract repeated markup into a component. Extract repeated rendering logic into a hook or function.
- Prefer composition (`children`, slots) over configuration props. A component with ten boolean props needs to be several components.
- Keep copy (labels, messages) in the component while it is used once. Move it to a constant only when reused. *(opinion)*

## 3.3 State

- Keep state minimal. Derive everything you can during render.

```tsx
const hasChanges = draft.name !== saved.name || draft.qty !== saved.qty;
```

- One source of truth. Do not copy props or query data into state unless the user edits a draft of it.
- Forms keep a local draft and write to the backend once, on save.
- Keep state near where it is used. Lift it only when siblings need it, and use context only when passing through many levels.
- Use a reducer when several fields change together or transitions have rules.
- Model UI modes with a single union (`'idle' | 'editing' | 'saving'`), not several booleans.
- Server state (data from an API) is not UI state. Use a data-fetching library that handles caching, invalidation and loading, and do not mirror its results into `useState`.
- After a mutation succeeds, write the response into the cache and invalidate what depends on it. Do not refetch what you already have.

## 3.4 Effects

- An effect is for synchronizing with something outside React: subscriptions, timers, storage, imperative APIs. If you can compute it during render or in an event handler, do that.
- Do not use an effect to derive state from props or state. Compute it directly.
- Do not use an effect to respond to a user action. Put the logic in the event handler.
- Every effect cleans up after itself (unsubscribe, abort, clear timeout).
- Dependency arrays are complete and honest. If the linter complains, restructure: move the value inside, extract a function, or use a ref for a value that must not retrigger. Do not disable the rule.
- Guard against stale async results with an abort signal or an ignore flag.

```tsx
useEffect(() => {
  const controller = new AbortController();
  load(id, controller.signal).then(setOrder);
  return () => controller.abort();
}, [id]);
```

## 3.5 Hooks

- One hook, one responsibility, one file, named `useThing`.
- Three kinds of hooks, kept distinct. *(opinion)*

| Kind | Naming | Returns |
| --- | --- | --- |
| Read | `useGetX`, `useIsX`, `useX` | data or a boolean |
| Write | `useCreateX`, `useUpdateX`, `useDeleteX` | a named action and its pending state |
| Behavior | `usePreventX`, `usePollX` | nothing, side effect only |

- Return specific names, not generic ones: `{deleteOrder, isPending}`, not `{run, loading}`.
- Wrap library hooks and bind ambient values (current user, tenant, route ids) inside the wrapper so callers do not pass them.
- Compose small hooks into bigger ones. Do not grow one hook into a god hook that confirms, builds a payload, calls the API, updates the cache, notifies and navigates. Extract each step.
- A write hook owns its user feedback and cache updates so components only call the action. *(opinion)*

```tsx
export const useArchiveProject = () => {
  const {addNotice} = useNotices();
  const {mutateAsync, isPending} = useArchiveProjectMutation();

  const archiveProject = async (projectId: string) => {
    try {
      await mutateAsync({projectId});
      addNotice({type: 'success', message: 'Project archived'});
    } catch (error) {
      console.error(error);
      addNotice({type: 'error', message: 'Failed to archive project'});
    }
  };

  return {archiveProject, isPending};
};
```

- Return a result union (`'success' | 'cancelled' | 'error'`) from an action when the caller must react. Do not throw for expected outcomes such as the user cancelling a confirmation.
- Hooks follow the rules of hooks: top level only, no conditions, no loops.

## 3.6 Memoization and performance

- Do not memoize by default. `useMemo`, `useCallback` and `memo` add code and noise.
- Memoize when identity matters (a value passed to a memoized child or a library that compares by reference) or when work is measurably expensive.
- Measure before optimizing. Fix the real cause: too much state high in the tree, unstable context values, huge lists.
- Split context by change frequency. Use a selector-based store or context selector when consumers should not re-render on every change.
- Virtualize very long lists.
- Code-split by route or heavy feature, not by component.

## 3.7 Context and providers

- One file per provider containing the context, the provider and the accessor hook. *(opinion)*
- Keep the context value small and typed. Expose a hook (`useCart`), not the raw context.
- Do not put fast-changing values and stable values in the same context.
- Throw a clear error from the accessor when a required provider is missing, or provide a safe default deliberately.

## 3.8 Styling and UI

- Build from the project's design-system components. Layout comes from layout components (stacks, grids), not ad hoc CSS.
- Use one styling approach per project. Scoped styles (modules, CSS-in-JS, utility classes) for what components cannot express.
- Avoid inline styles beyond one or two properties.
- Semantic HTML first: real buttons for actions, links for navigation, labels for inputs, headings in order.
- Every interactive element is keyboard reachable and has an accessible name. Icon-only buttons have a label.
- Add a stable test id only where nothing accessible distinguishes the element.
- Handle every data state in the UI: loading, empty, error, success. Do not render only the happy path.

## 3.9 Forms

- Local draft state, validated with pure validator functions, submitted once.
- Disable or block save while there are validation errors, and show the reason, not just a disabled button.
- Show the unsaved-changes state (`hasChanges`) and guard against losing it (confirm on close or navigation).
- Show pending state on submit and prevent double submission.
- Reset or re-initialize the draft when the underlying record changes identity, using a `key` on the form rather than an effect.

## 3.10 Data fetching

- Data access goes through one layer (generated clients, query hooks). No raw `fetch` or HTTP calls scattered through components.
- Components receive data and states from hooks. They do not know URLs or payload shapes.
- Treat loading, error and empty as first-class outcomes of every read.
- Cache keys are derived from the parameters that define the data.

## 3.11 Testing React

Test what the user sees and does. Never test what the component is made of.

- Render the real component (or page) with its real hooks and providers. Fake only the outside world: the network, router, browser APIs.
- Fake the network with a request-interception tool and a small stateful fake backend, so create, update and delete really change what the next read returns.
- Interact as a user would: click, type, select, tab. Prefer the user-event library over low-level event dispatch.
- Query the way a user finds things, in this order:
  1. by role and accessible name
  2. by label text
  3. by visible text
  4. by test id, only when nothing accessible distinguishes the element
- Use asynchronous queries (`findBy*`) for anything that appears later. Do not wrap synchronous queries in `waitFor`.
- Do not use `container.querySelector`, CSS class selectors, snapshots, component instance access or fixed sleeps.
- Assert on what appears: text on screen, a control's state, an item in a list.
- Do not mock your own hooks, context providers or child components. Put the app in the state you need by using the fake backend or real interaction.
- Assert on a mocked function only for a boundary effect the user cannot see, such as a navigation.
- Cover loading, empty, error and validation states by overriding the fake network for that test.
- Shared setup becomes a helper named after the user's goal (`openEditor`, `addItemToCart`) that waits for its own completion.

```tsx
it('allows renaming a project', async () => {
  const {user} = render(<ProjectPage />);

  await user.click(await screen.findByRole('button', {name: 'Rename'}));
  await user.clear(screen.getByRole('textbox', {name: 'Name'}));
  await user.type(screen.getByRole('textbox', {name: 'Name'}), 'Roadmap');
  await user.click(screen.getByRole('button', {name: 'Save'}));

  expect(await screen.findByText('Project renamed')).toBeInTheDocument();
  expect(screen.getByRole('heading', {name: 'Roadmap'})).toBeInTheDocument();
});

it('shows an error when saving fails', async () => {
  server.use(http.put('*/projects/:id', () => HttpResponse.error()));
  const {user} = render(<ProjectPage />);

  await user.click(await screen.findByRole('button', {name: 'Rename'}));
  await user.click(screen.getByRole('button', {name: 'Save'}));

  expect(await screen.findByText('Failed to rename project')).toBeInTheDocument();
});
```

- Pure logic extracted from components (formatting, validation, derivation) gets its own direct tests. That is usually the cheapest way to cover rules.
- Test hooks through the component that uses them. Test a hook alone only when it is a reusable primitive with its own contract.

## 3.12 React review checklist

- [ ] Components render. Logic lives in hooks or plain functions.
- [ ] No derived state stored in `useState`. No effect that could be a computation or an event handler.
- [ ] Every effect cleans up. No disabled exhaustive-dependency rule.
- [ ] No nested ternaries. No components defined inside components. Stable keys on lists.
- [ ] Loading, empty and error states are handled.
- [ ] No unnecessary `useMemo` or `useCallback`.
- [ ] Interactive elements are semantic, labelled and keyboard reachable.
- [ ] Tests interact like a user, query by role or label, and mock only the network and platform boundary.

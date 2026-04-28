# Angular 20 + TypeScript Code Style

Frontend apps target **Angular 20** on **TypeScript 5.5+**, packaged with the **Angular CLI** and the new **application builder** (esbuild). This document is what reviewers will hold you to.

**Tooling enforces what it can:** `eslint` (with `@angular-eslint`, `@typescript-eslint`), `prettier`, `tsc --noEmit` strict, `stylelint`, `ng test` (Karma+Jasmine *or* Jest), `playwright`, `axe-core`. Things tooling can't enforce are below.

---

## 1. Project shape

- **Standalone everything.** No `NgModule` — Angular 20's `bootstrapApplication` is the only entry point.
- **Strict mode on, no exceptions:**
  ```jsonc
  // tsconfig.json
  "strict": true,
  "noUncheckedIndexedAccess": true,
  "noImplicitOverride": true,
  "noFallthroughCasesInSwitch": true,
  "exactOptionalPropertyTypes": true,
  // angular.json compilerOptions
  "strictTemplates": true,
  "strictInjectionParameters": true,
  "strictInputAccessModifiers": true
  ```
- **Folder layout** — feature-first:
  ```
  src/app/
    core/                 ← cross-cutting singletons: api client, auth, interceptors, guards
    shared/               ← reusable, dependency-free UI: primitives, pipes, directives, tokens
    layout/               ← shell, header, sidebar, navigation
    features/
      <feature>/
        <feature>.routes.ts
        <feature>.service.ts
        pages/            ← route-level components
        components/       ← feature-local presentational components
        models/           ← types & schemas
    app.config.ts         ← providers, route-level config
    app.routes.ts
    main.ts
  ```
- **Nx / monorepo** is preferred for ≥3 apps. One `lib` per feature, enforced module boundaries via `@nx/eslint-plugin`.

---

## 2. Naming

| Kind | Convention | Example |
|---|---|---|
| File | `kebab-case` | `invoice-detail.component.ts` |
| Class | `PascalCase` | `InvoiceDetailComponent` |
| Selector | `kebab-case`, `<prefix>-` | `<acme-invoice-detail>` |
| Function / variable | `camelCase` | `loadInvoice` |
| Type / Interface | `PascalCase`, **no `I` prefix** | `Invoice`, `InvoiceFilter` |
| Enum value | `PascalCase` | `InvoiceStatus.Draft` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_PAGE_SIZE` |
| Signal | no suffix | `invoice` |
| Observable | suffix `$` | `invoice$` |
| InjectionToken | `UPPER_SNAKE_CASE` | `API_BASE_URL` |

File suffixes (Angular convention, lowercase):
`*.component.ts`, `*.service.ts`, `*.directive.ts`, `*.pipe.ts`, `*.guard.ts`, `*.resolver.ts`, `*.interceptor.ts`, `*.routes.ts`, `*.model.ts`, `*.store.ts`, `*.spec.ts`.

Avoid:
- Hungarian or type prefixes (`strName`, `IUser`).
- Generic nouns: `Helper`, `Util`, `Manager`, `Handler` — name the actual concept.
- `data`, `info`, `result`, `obj` — use the domain term.
- Single-letter names except trivially scoped iterators.

---

## 3. Components

- **Standalone with explicit `imports`.** No re-exports through "shared modules."
- **`OnPush` change detection on every component.** Default change detection is forbidden in new code.
- **Signal-based inputs/outputs/queries** — `input()`, `input.required()`, `output()`, `viewChild()`, `contentChild()`. The decorator-based `@Input` / `@Output` / `@ViewChild` are forbidden in new code.
- **`inject()` for dependencies.** Constructor injection allowed only when a base class needs it.
- **Templates ≤200 lines.** Larger = decompose into child components.
- **Logic-free templates.** Move conditions into `computed()` or methods. Pipes are fine; method calls in bindings are not (re-evaluated on every CD cycle).
- **Built-in control flow only:** `@if`, `@for`, `@switch`, `@defer`. `*ngIf` / `*ngFor` / `*ngSwitch` are forbidden.
- **`track` is mandatory in `@for`** — use a stable identity, not `$index`.
- **`host` in the `@Component` metadata**, not `@HostBinding` / `@HostListener` decorators.
- **Access modifiers in templates:** template-only state is `protected readonly`; private state is `private`; public surface is `readonly` only when consumed by parents.

```ts
import { ChangeDetectionStrategy, Component, computed, inject, input } from '@angular/core';
import { InvoiceService } from '../invoice.service';

@Component({
  selector: 'acme-invoice-summary',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  templateUrl: './invoice-summary.component.html',
  styleUrl: './invoice-summary.component.scss',
  host: { 'class': 'acme-invoice-summary', '[attr.data-status]': 'status()' },
})
export class InvoiceSummaryComponent {
  readonly invoiceId = input.required<string>();

  private readonly invoices = inject(InvoiceService);

  protected readonly invoice = this.invoices.detail(this.invoiceId);
  protected readonly status   = computed(() => this.invoice()?.status ?? 'unknown');
}
```

---

## 4. State management

- **Signals first.** `signal()`, `computed()`, `effect()`, `linkedSignal()`, `resource()` cover most needs.
- **`resource()` / `httpResource()`** for read state tied to an input — handles loading/error/refresh out of the box. Prefer it over manual `subscribe` + signal plumbing.
- **RxJS at boundaries only:** HTTP responses, SSE, WebSockets, router events. Convert to signals via `toSignal()` for template consumption.
- **No NgRx / NGXS / Akita** unless an ADR justifies it. Per-feature signal stores cover what we need; reach for a store library only when multiple features share mutable state.
- **Service-with-signals = the default store pattern**:
  ```ts
  @Injectable({ providedIn: 'root' })
  export class InvoiceStore {
    private readonly _selected = signal<InvoiceId | null>(null);
    readonly selected = this._selected.asReadonly();
    select(id: InvoiceId) { this._selected.set(id); }
  }
  ```
- **`effect()` is a last resort.** Most reactive flows compose with `computed()` / `linkedSignal()`. Use `effect()` only for genuine side effects (logging, DOM imperative ops, navigation).

---

## 5. Services

- **`@Injectable({ providedIn: 'root' })`** for app-wide services. **Never** `providedIn: 'platform'` — it's almost always wrong.
- **One responsibility per service.** `InvoiceService` does invoices. Don't bolt payment-management onto it.
- **HTTP via the generated API client only** (`core/api/generated/`). Never raw `HttpClient` calls in features.
- **No business logic in components** when it can live in a service. Components orchestrate; services compute.

---

## 6. HTTP / API

- **API client is generated** from the backend OpenAPI spec (`openapi-generator` or `orval`). Never edit `core/api/generated/` by hand.
- **`HttpInterceptor`s for cross-cutting:** auth (cookie + CSRF or bearer), correlation-id propagation, error-envelope mapping, retry/backoff.
- **`provideHttpClient(withFetch(), withInterceptors([...]))`** — the fetch backend is the default; XHR only for legacy needs.
- **Errors mapped to typed `ApiError`** in the interceptor. Components consume typed errors, not raw `HttpErrorResponse`.
- **SSE** via a thin `SseClient` returning an Observable; consumers convert to a signal as needed.
- **Never call `.subscribe()` in a component.** Convert to signal with `toSignal()` or use the `async` pipe — let the framework manage the subscription.

---

## 7. Forms

- **Reactive Forms only.** No template-driven forms.
- **Strictly typed forms:** `FormGroup<{ name: FormControl<string>; ... }>` with explicit type parameters.
- **Validation messages from a central registry** keyed by error code. Never inline strings in templates.
- **Submit disabled while pending or invalid.** Always show a spinner on the submit button while the request is in flight.
- **Form state is a signal** — derive `disabled`, `errors`, `pending` via `toSignal(form.statusChanges)` and surface them in the template.

---

## 8. Routing

- **Lazy-load every feature** via `loadChildren: () => import('./features/invoices/invoices.routes')`.
- **Functional guards / resolvers / interceptors** — `CanActivateFn`, `ResolveFn`, `HttpInterceptorFn`. Class-based guards are forbidden in new code.
- **Resolvers for required data** before route activation when the empty-loading state would be jarring (e.g., detail pages).
- **Query params for filter state.** Pagination, status filters, search — all in the URL so links are shareable.
- **Use `withComponentInputBinding()`** so route params bind directly to component `input()`s.

---

## 9. TypeScript discipline

- **`any` is forbidden** without a comment justifying the escape. ESLint enforces.
- **Prefer `unknown` over `any`** at boundaries you can't fully type.
- **`readonly` everywhere it's true** — fields, arrays (`readonly T[]`), tuples.
- **`as const`** for literal lookup tables and discriminated-union literals.
- **Discriminated unions over enums** for closed sets of states. Numeric `enum` is forbidden; `const enum` is forbidden (breaks isolated modules); a `PascalCase` string-enum or `as const` object is fine.
- **Type predicates / assertion functions** (`x is Foo`) at parse boundaries.
- **`zod`** (or equivalent) to validate any external payload that isn't covered by a generated client.
- **No `non-null assertion (`!`)** without a comment. The bang silences the compiler; readers need the reason.

---

## 10. Styling

- **SCSS only.** No CSS-in-JS, no styled-components, no inline `style` strings beyond truly dynamic values.
- **Design tokens in `shared/styles/tokens.scss`** — single source of truth. Never hardcode colors, spacing, or font sizes.
- **Component styles are scoped** by Angular's default emulated encapsulation. `::ng-deep` is forbidden in new code.
- **Class naming follows BEM-lite primitives**: `acme-card`, `acme-card__header`, `acme-card--compact`. Variants via modifier classes.
- **Layout via CSS Grid / Flexbox.** No table layouts, no floats, no absolute positioning for layout flow.
- **CSS variables for theming.** Token files emit CSS custom properties; runtime theming flips them.
- **Responsive breakpoints from tokens.** Don't write magic-number media queries in components.
- **`stylelint`** runs in CI and matches Prettier formatting.

---

## 11. Accessibility

- **WCAG 2.2 AA** is the bar. `axe-core` runs in component tests; new violations fail the PR.
- **All interactive elements keyboard-accessible.** Tab order matches visual order; never `tabindex` >0.
- **`aria-*` on custom controls.** Modals trap focus; dropdowns expose `aria-expanded` / `aria-controls`; live regions use `aria-live`.
- **Color is never the only signal.** Status uses icon + color; error fields use border + text.
- **Focus management** on route changes — set focus to the page heading, restore focus on dialog close.
- **CDK a11y primitives** (`@angular/cdk/a11y`) preferred over hand-rolled focus traps.

---

## 12. Performance

- **`@for` `track` is mandatory** — by stable id, never `$index` for keyed lists.
- **`@defer` for heavy components** — charts, rich editors, modals. Use `on idle`, `on viewport`, or `on interaction` triggers.
- **Lazy-load images** with `loading="lazy"` and the `NgOptimizedImage` directive. `priority` only on the LCP image.
- **No method calls in templates** that aren't pure and trivial — they re-run on every CD cycle.
- **`async` pipe over manual subscribe**, but prefer `toSignal()` in OnPush components.
- **Bundle budgets** in `angular.json` — fail the build if a route exceeds 250 KB gzipped without an exception.
- **Server-side rendering** (`@angular/ssr`) with hydration enabled by default for any user-facing app; opt-out per-route only when needed.

---

## 13. Testing

Full guide in `TESTING.md`. Highlights:

- **Unit tests with Jest** (or Karma+Jasmine, pick one per repo). `TestBed.configureTestingModule({ providers: [...] })` with mocked dependencies.
- **`provideRouter([])`, `provideHttpClientTesting()`** — the modern functional providers, not legacy modules.
- **Signal testing:** assert via `fixture.componentInstance.signal()`; mutate via `set()` / `update()` and call `fixture.detectChanges()`.
- **Component test files live next to the component** — `foo.component.spec.ts`.
- **Query the DOM via `data-testid`** attributes (Angular Testing Library style). Class-name selectors are forbidden in tests.
- **No `fakeAsync` / `tick`** unless the test genuinely depends on virtual time. Prefer `await fixture.whenStable()`.
- **Playwright for E2E.** No Cypress in new repos.
- **Coverage floor: 80% line, 70% branch** on services and shared components.

---

## 14. Imports & module boundaries

- **No barrel `index.ts` files in features** — they hurt tree-shaking and create import cycles. `core/` and `shared/` may have small barrels for ergonomics; features import direct paths.
- **`@app/*` path alias** for `src/app/*` — configure in `tsconfig.json`. No `../../../` chains.
- **Imports ordered:** Angular → third-party → `@app/...` → relative. Prettier + `eslint-plugin-import` enforce.
- **Module boundary rules** (Nx `enforce-module-boundaries` or `eslint-plugin-boundaries`): `features` cannot import other `features`; `shared` cannot import `features`; `core` cannot import `features`.

---

## 15. Internationalization

- **`@angular/localize`** with extracted XLIFF; runtime translations via `$localize` template-literal tags.
- **Never concatenate translated strings.** Use ICU message format for plurals/genders.
- **Date / number formatting** through `DatePipe` / `DecimalPipe` / `Intl.*` — never hand-rolled.
- **Right-to-left tested** for any app shipping to RTL locales.

---

## 16. Comments and docs

- **Default to no comments.** TypeScript types and good names cover most cases.
- **JSDoc only on the public API of services / utilities** intended for cross-feature consumption.
- **No commented-out code.** Delete it; git remembers.
- **No `// TODO` without a linked ticket.**
- **Never reference the current PR or task** — those rot.

---

## 17. Don'ts (review will reject)

- `any` without a comment explaining why.
- `// @ts-ignore` / `// @ts-expect-error` without a linked ticket number.
- `console.log` in production code. Use the configured logger / `ToastService` for user-visible messages.
- `setTimeout` to "wait for change detection" — use `fixture.whenStable()` or signals.
- Direct DOM manipulation (`document.querySelector`) outside narrowly-scoped directives.
- `NgModule` in new code.
- `*ngIf`, `*ngFor`, `*ngSwitch` — use the `@`-control-flow.
- `@Input` / `@Output` decorators — use `input()` / `output()`.
- Constructor injection of services in new components — use `inject()`.
- Importing from `rxjs/internal/...` — not part of the public API.
- Editing generated client code.
- `providedIn: 'platform'`.
- Mixing CommonJS and ES modules.
- `enum` (numeric) — use string union types or `as const` objects.
- Mutating an input signal's value directly (treat `input()` as readonly).

---

## 18. Recommended reading

- [Angular Style Guide](https://angular.dev/style-guide) — the official one.
- [Angular Signals](https://angular.dev/guide/signals) and [Resource API](https://angular.dev/guide/signals/resource).
- [Angular Testing Library](https://testing-library.com/docs/angular-testing-library/intro) — the user-centric testing approach.
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html) — chapters on narrowing, generics, and utility types.
- [web.dev Performance](https://web.dev/learn/performance) — the core-web-vitals primer.
- [WCAG 2.2 Quick Reference](https://www.w3.org/WAI/WCAG22/quickref/) — bookmark it.

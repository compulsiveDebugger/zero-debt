# Angular / TypeScript Code Style

The frontend (`web/`) is Angular 20 + TypeScript 5+. This document is what reviewers will hold you to.

**Tooling enforces what it can:** `eslint`, `prettier`, `tsc --noEmit` strict, `ng test`, `ng e2e`. The rest is below.

---

## 1. Project shape

- **Standalone components throughout** — no `NgModule` except the bootstrap module.
- **Strict mode on:** `"strict": true`, `"strictTemplates": true`, `"strictInjectionParameters": true` in `tsconfig.json` and `angular.json`.
- **Folder layout:**
  ```
  web/src/app/
    core/           ← cross-cutting: api client, auth, interceptors, guards
    shared/         ← reusable UI: shell, chrome, primitives, pipes, tokens
    features/       ← feature modules: auth, repos, runs, settings
      <feature>/
        <feature>.component.ts
        <feature>.routes.ts
        <feature>.service.ts
        components/
        models/
  ```

---

## 2. Naming

| Kind | Convention | Example |
|---|---|---|
| File | `kebab-case` | `repo-detail.component.ts` |
| Class | `PascalCase` | `RepoDetailComponent` |
| Selector | `kebab-case` w/ `zd-` prefix | `<zd-repo-detail>` |
| Function / variable | `camelCase` | `connectRepo` |
| Type / Interface | `PascalCase` (no `I` prefix) | `RepoConnection` |
| Enum value | `PascalCase` | `RunStatus.Partial` |
| Constant | `UPPER_SNAKE_CASE` | `MAX_REPOS_PER_PAGE` |
| Observable | suffix `$` | `currentUser$` |
| Signal | no suffix | `currentUser` |

File suffixes (Angular convention):

- `*.component.ts`, `*.service.ts`, `*.directive.ts`, `*.pipe.ts`, `*.guard.ts`, `*.resolver.ts`, `*.interceptor.ts`, `*.routes.ts`, `*.model.ts`.

---

## 3. Components

- **Standalone** with explicit `imports`.
- **`OnPush` change detection** for every component. Default change detection is forbidden.
- **Inputs / outputs** declared via the `input()` / `output()` signal-based APIs (not the legacy `@Input()` / `@Output()` decorators).
- **Inject dependencies via `inject()`**, not constructor injection.
- **Templates ≤200 lines.** Larger = decompose into child components.
- **Logic-free templates.** Move conditions into computed signals or methods.
- **No `*ngIf` / `*ngFor`** — use the new control-flow syntax (`@if`, `@for`, `@switch`).

```ts
import { ChangeDetectionStrategy, Component, inject, input } from '@angular/core';
import { RunsService } from './runs.service';

@Component({
  selector: 'zd-run-summary',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  templateUrl: './run-summary.component.html',
  styleUrl: './run-summary.component.scss',
})
export class RunSummaryComponent {
  readonly runId = input.required<string>();
  private readonly runs = inject(RunsService);
  protected readonly run = this.runs.runDetail(this.runId);
}
```

---

## 4. State management

- **Signals first.** Use `signal()`, `computed()`, `effect()` for component and service state.
- **Observables (RxJS)** at boundaries: HTTP, SSE, router. Convert to signals via `toSignal()` for template consumption.
- **No NgRx / Akita** unless an explicit ADR justifies it. Per-feature services with signals cover our needs.
- **Avoid global mutable singletons** beyond `AuthService` and `ToastService`.

---

## 5. Services

- **`@Injectable({ providedIn: 'root' })`** for app-wide services; **`providedIn: 'platform'`** never (we don't need it).
- **One responsibility per service.** `ReposService` does repos. Don't bolt run-management onto it.
- **HTTP via the generated API client only** (`web/src/app/core/api/generated/`). Never raw `HttpClient` calls in features.
- **No business logic in components** when it can live in a service. Components orchestrate; services compute.

---

## 6. HTTP / API

- **API client is generated** from the backend OpenAPI spec. Never edit `core/api/generated/` by hand.
- **`HttpInterceptor`s** for cross-cutting: auth (cookie + CSRF), error mapping, request id propagation.
- **Errors mapped to typed `ApiError`** in the interceptor. Components consume typed errors, not raw `HttpErrorResponse`.
- **SSE via `EventSource`** wrapped in a thin `SseClient` that returns an Observable; consumers convert to signal as needed.

---

## 7. Forms

- **Reactive Forms only.** No template-driven forms.
- **Typed forms:** `FormGroup<{...}>` with explicit type parameter.
- **Validation messages from a central registry** keyed by error code; never inline strings in templates.
- **Submit disabled while pending or invalid.** Always show a spinner on the submit button while the request is in flight.

---

## 8. Routing

- **Lazy-load every feature.** `loadComponent: () => import('./features/runs/...')`.
- **Route-level guards** for auth and authorization.
- **Resolvers for required data** before route activation when the empty-loading state would be jarring (e.g., run detail).
- **Query params for filter state.** Pagination, status filters, search — all in the URL so links are shareable.

---

## 9. Styling

- **SCSS only.** No CSS-in-JS, no styled-components.
- **Design tokens in `shared/styles/tokens.scss`** — single source of truth. Never hardcode colors / spacing.
- **Component styles scoped** by Angular's view encapsulation default (Emulated). Never `::ng-deep` in new code.
- **Class naming follows the mockup primitives:** `zd-card`, `zd-chip`, `zd-phase-tag`, etc. Variants via modifier classes (`zd-chip--ok`).
- **Layout via CSS Grid / Flexbox.** No table layouts. No floats.
- **Responsiveness:** desktop-first, min 1024px. Side nav collapses below that.

---

## 10. Accessibility

- **All interactive elements keyboard-accessible.** Tab order matches visual order.
- **`aria-*` on custom controls.** Modals trap focus; dropdowns expose `aria-expanded`.
- **Color is never the only signal.** Status uses icon + color; error fields use border + text.
- **Run `axe-core`** in component tests for any new view; PR fails if it introduces new violations.

---

## 11. Testing

Full guide in [TESTING.md](TESTING.md). Highlights:

- **`TestBed.configureTestingModule({ providers: [...] })`** with mocked services.
- **Use signals' `set()` / `update()`** in tests; assert via `fixture.componentInstance.signal()`.
- **Component test files live next to the component** — `foo.component.spec.ts`.
- **No DOM querySelectors with class names from templates** — use `data-testid="..."` attributes (Angular Testing Library style).

---

## 12. Imports

- **No barrel `index.ts` files** in features — they hurt tree-shaking. `core/` and `shared/` may have small barrels for ergonomics; features import direct paths.
- **`@app/*` path alias** for `src/app/*` — configure in `tsconfig.json`. No `../../../` import chains.
- **Imports ordered:** Angular → third-party → `@app/...` → relative. Prettier enforces.

---

## 13. Performance

- **`trackBy` on every `@for`** when iterating > ~10 items. Required for runs/issue tables.
- **Lazy load images / heavy components** with `@defer` blocks.
- **Avoid `async` pipe in tight loops.** Subscribe once at the top, project to signals for templates.
- **No new RxJS operators imported** when an existing one would do — bundle size matters.

---

## 14. Comments and docs

- **Default to no comments.** TypeScript types and good names cover most cases.
- **JSDoc only for the public API** of services intended for cross-feature consumption.
- **No commented-out code.** Delete it; git remembers.

---

## 15. Don'ts (review will reject)

- `any` type without a comment explaining why.
- `// @ts-ignore` / `// @ts-expect-error` without a follow-up issue number.
- `console.log` in production code (a custom logger goes through `ToastService` for user-visible messages).
- `setTimeout` for "wait for change detection" — use `fixture.whenStable()`.
- Direct DOM manipulation (`document.querySelector`) outside narrowly-scoped directives.
- `NgModule` in new code.
- Mixing CommonJS and ES modules.
- Importing from `rxjs/internal/...` — not part of the public API.
- Editing generated client code.

---

## 16. Recommended reading

- [Angular Style Guide](https://angular.dev/style-guide) — the official one.
- [Angular Signals docs](https://angular.dev/guide/signals).
- [Angular Testing Library](https://testing-library.com/docs/angular-testing-library/intro) — the user-centric testing approach.
- [TypeScript Deep Dive](https://basarat.gitbook.io/typescript/) — for the rare areas tooling doesn't cover.

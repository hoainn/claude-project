# Code Style

## Angular / TypeScript

- Standalone components (implicit `standalone: true` in Angular 18+)
- OnPush change detection strategy
- Prefer `input()` / `output()` functions over `@Input()` / `@Output()` decorators
- Prefer `inject()` function over constructor injection
- Use new control flow syntax: `@if`, `@for`, `@switch` — not `*ngIf`, `*ngFor`
- Use signals for local state; `computed()` for derived state
- Smart/dumb component pattern; lazy load modules and components
- NgRx functional approach: `createFeature`, `createReducer`, `createActionGroup`
- Single quotes for strings; `const` preferred; avoid `any` — use `unknown` or utility types
- Filenames: kebab-case — `*.component.ts`, `*.service.ts`, `*.spec.ts`
- Memory leaks: always use `async` pipe, `takeUntil`, or `untilDestroyed(this)`

### Import order
1. Angular core/common
2. RxJS
3. Angular-specific modules
4. Core application imports
5. Shared module imports
6. Environment imports
7. Relative path imports

## CSS / SCSS

- REM units (not PX)
- All colors defined in `styles/_colors.scss`
- Avoid `::ng-deep` and `!important`
- Dashes over camelCase in class names
- Use `[class.active]` not `[ngClass]` for single class bindings

## C# / .NET Backend

- CQRS with MediatR — commands and queries in `Redactable.Core`
- Entity Framework for data access in `Redactable.Data`
- Domain models in `Redactable.Domain`
- REST API controllers in `Redactable.API`
- Stripe billing in `Redactable.Billing.Stripe`

## Testing

- Arrange-Act-Assert pattern
- Test happy and unhappy paths
- Unit test files: `*.spec.ts`
- E2E tests tagged: `Happy`, `Smoke`, `Regression`
- Frontend: Jest (unit), Cypress (E2E)
- Batch frontend: Vitest (unit), Cypress (E2E)
- Backend: integration tests in `Redactable.IntegrationTests`

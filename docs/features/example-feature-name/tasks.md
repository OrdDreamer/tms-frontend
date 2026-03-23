# EXAMPLE — Feature Tasks Template

> Це демонстраційний приклад `tasks.md`. Заміни приклади на реальні задачі під конкретну фічу.

---

## Feature Link

- Spec: `/docs/features/feature-name/spec.md`

## Checklist (атомарні задачі)

- [ ] Task 1: Add bulk edit trigger in translations toolbar
  - Input:
    - spec: `/docs/features/feature-name/spec.md`
  - Output:
    - toolbar button visible only when 1+ keys selected
    - button opens bulk edit side panel
  - Constraints:
    - reuse existing UI patterns from `/docs/standards/ui-patterns.md`
    - no inline styles

- [ ] Task 2: Implement bulk edit panel form
  - Input:
    - spec: `/docs/features/feature-name/spec.md`
  - Output:
    - `@mantine/form` with fields: target language, value
    - submit button supports loading/disabled states
    - validation errors rendered near fields
  - Constraints:
    - local form state only
    - field errors from `getFieldErrors`

- [ ] Task 3: Add mutation flow for applying changes to selected keys
  - Input:
    - spec: `/docs/features/feature-name/spec.md`
  - Output:
    - mutation hook in `features/*/api`
    - on success invalidates `translationKeyKeys.lists(projectSlug)`
    - shows success feedback
  - Constraints:
    - no broad cache invalidation
    - no direct axios calls from UI components

- [ ] Task 4: Handle edge states and error scenarios
  - Input:
    - spec: `/docs/features/feature-name/spec.md`
  - Output:
    - explicit handling for empty selection, partial fail, server error
    - no custom 401 retry logic in component
  - Constraints:
    - follow state rules from `/docs/architecture/state-management.md`

- [ ] Task 5: Add tests for critical flow
  - Input:
    - spec: `/docs/features/feature-name/spec.md`
  - Output:
    - test: successful bulk update invalidates list and closes panel
    - test: validation error shown near field
  - Constraints:
    - test files colocated (`*.test.ts(x)`)
    - use shared MSW handlers where relevant

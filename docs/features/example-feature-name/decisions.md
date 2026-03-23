# EXAMPLE — Decision Log Template

> Це демонстраційний приклад `decisions.md`. Фіксує не "що зроблено", а "чому саме так".

---

## Decision 001: Bulk edit via side panel, not inline multi-row editing

- Status: accepted
- Why:
  - side panel дає стабільний простір для валідації та помилок
  - inline multi-row редактор суттєво ускладнює UX і тестування
- Impact:
  - швидше впровадження
  - менше ризику регресій у таблиці

## Decision 002: Use standard mutation flow + precise invalidation

- Status: accepted
- Why:
  - узгоджено зі `docs/architecture/state-management.md`
  - точкова інвалідація зменшує зайві перезапити
- Impact:
  - стабільна синхронізація UI після submit
  - передбачувана продуктивність

## Decision 003: Keep form state local with @mantine/form

- Status: accepted
- Why:
  - відповідає правилам форми з `docs/standards/coding-guidelines.md`
  - не створює зайвого глобального state
- Impact:
  - простіша підтримка
  - чистіші межі відповідальності

## Decision 004: No local 401 handling in feature components

- Status: accepted
- Why:
  - 401/retry flow уже централізовано в axios interceptor
- Impact:
  - менше дублювання логіки
  - одна поведінка auth-помилок у всьому застосунку

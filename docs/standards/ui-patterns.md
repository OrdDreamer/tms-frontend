# UI Patterns

Operational guide для консистентної реалізації UI у проєкті. Документ фіксує повторювані патерни взаємодії, стани інтерфейсу та поведінку компонентів.

> Canonical-рішення щодо архітектури, state-flow і API залишаються в `docs/architecture/*`. Якщо є конфлікт, пріоритет має architecture-документація.

---

## Scope

- Документ регулює **поведінку UI** і повторювані interaction patterns.
- Документ не перевизначає FSD-структуру, auth flow або API contracts.
- Пов'язані документи:
  - `docs/architecture/frontend-architecture.md`
  - `docs/architecture/state-management.md`
  - `docs/architecture/api-contracts.md`
  - `docs/standards/coding-guidelines.md`

---

## 1) Базові UI-принципи

### Rule: Однакова дія — однаковий control pattern
**Why:** користувач не перевчається між екранами.

**Example**
- primary submit: однаковий стиль кнопки на формах
- destructive action: завжди через confirm-step

### Rule: Система завжди дає видимий фідбек
**Why:** користувач розуміє, що відбувається після дії.

**Example**
- loading indicator під час async-операції
- success/error feedback після submit

### Rule: A11y не опційна
**Why:** керування з клавіатури та читабельність для assistive tech.

**Example**
- semantic labels для полів
- керований focus у модалках
- доступність кнопок через зрозумілий текст/aria-label

---

## 2) Page State Pattern

### Rule: Кожна сторінка/ключовий віджет має явно обробляти `loading`, `empty`, `error`, `success`
**Why:** передбачуваний UX, без "порожніх" або двозначних станів.

**Example**
```tsx
if (isLoading) return <PageLoader />;
if (isError) return <ErrorState />;
if (!data?.length) return <EmptyState />;
return <Content data={data} />;
```

`isError` / `ErrorState` — це **збій завантаження даних** (мережа, API). **Неконтрольовані помилки рендеру** (throw у компоненті) має перехоплювати глобальний **Error Boundary** у `app/` — див. [Frontend Architecture → Error Handling](../architecture/frontend-architecture.md#error-handling).

### Rule: `empty` state містить наступну дію (CTA), якщо сценарій дозволяє
**Why:** користувач не застрягає в тупиковому екрані.

---

## 3) Tables and Lists Pattern

### Rule: Для завантаження списків використовувати skeleton або явний loading-state
**Why:** краща сприйнятливість продуктивності.

### Rule: Пагінація синхронізується з URL (`page`, `limit`)
**Why:** back/forward коректні, URL можна поширити.

### Rule: Bulk actions виконуються лише після явного підтвердження
**Why:** захист від випадкових масових змін.

**Example**
- bulk delete відкриває confirm modal
- confirm блокується поки триває mutation

### Rule: Після bulk/single mutation оновлення списку через query invalidation
**Why:** консистентні дані без ручного "латання" UI.

---

## 4) Form Pattern

### Rule: Form state локальний через `@mantine/form`
**Why:** форми ізольовані, не засмічують глобальний state.

### Rule: Submit-кнопка має стани `idle/loading/disabled`
**Why:** уникнення подвійного submit і неясних взаємодій.

### Rule: Серверні field errors мапляться на поля через `getFieldErrors`
**Why:** точний і зрозумілий фідбек користувачу.

**Example**
```ts
onError: (error) => {
  const fieldErrors = getFieldErrors(error);
  Object.entries(fieldErrors).forEach(([field, messages]) => {
    form.setFieldError(field, messages[0]);
  });
}
```

### Rule: Unsaved changes мають явну поведінку
**Why:** користувач не втрачає введені дані випадково.

**Pattern**
- при close/cancel показати confirm, якщо форма dirty
- при успішному submit dirty-стан скидається

---

## 5) Async Interaction Pattern

### Rule: Глобальні API-помилки не дублювати локальними toasts без потреби
**Why:** уникнення подвійних повідомлень.

### Rule: Локальний `onError` використовувати для UI-специфіки
**Why:** поле/контрол отримує релевантний локальний фідбек.

### Rule: Optimistic update дозволений лише у сценаріях, де це зафіксовано
**Why:** зайва оптимістика підвищує ризик неконсистентності.

**У цьому проєкті**
- дозволено для inline-редагування перекладів
- інші мутації: стандартний success + invalidate flow

### Rule: Після 401 UI не реалізує власний retry-flow
**Why:** refresh/retry централізовано в axios interceptor.

---

## 6) Modal and Destructive Actions Pattern

### Rule: Модалка має очевидні `Confirm` і `Cancel`
**Why:** знижує ризик фатальних дій.

### Rule: `Confirm` блокується під час pending mutation
**Why:** запобігає дубльованим запитам.

### Rule: Закриття модалки при помилці не виконується автоматично
**Why:** користувач має побачити помилку і виправити стан.

---

## 7) URL-Driven UI Pattern

### Rule: Filter/sort/page/view state тримати в URL, не у глобальному store
**Why:** deep-linking, shareability, правильна історія навігації.

### Rule: Query key включає URL-параметри
**Why:** дані відповідають поточному стану URL.

**Example**
```ts
queryKey: translationKeyKeys.lists(projectSlug, { search, lang, status, page, limit })
```

---

## 8) i18n and RTL Pattern

### Rule: UI-тексти тільки через i18n ключі
**Why:** стабільна локалізація без hardcoded рядків.

### Rule: Для RTL мов на input встановлювати `dir="rtl"`
**Why:** коректна поведінка полів для `ar`, `he`, `fa`, `ur`.

### Rule: i18n key ідентифікує домен і дію
**Why:** ключі залишаються зрозумілими при масштабуванні.

**Example**
- `project.create_success`
- `translations.bulk_delete_confirm`

---

## 9) UI Anti-Patterns

- показувати порожній екран без `loading/empty/error` станів
- змішувати form state з global state без потреби
- закривати destructive modal без confirm
- дублювати глобальні toasts локальними повідомленнями
- зберігати filter/page/view state у Zustand замість URL
- робити optimistic updates у всіх мутаціях "за замовчуванням"
- хардкодити UI-рядки замість i18n ключів

---

## 10) UI Review Checklist

Перед merge перевір:

- у фічі явно покриті `loading/empty/error/success` стани
- форми мають submit/loading/disabled + field errors mapping
- destructive actions мають confirm-flow
- фільтри/пагінація/вид зберігаються в URL
- немає локальної логіки refresh token або дублювання error handling
- всі тексти проходять через i18n, RTL-поля поводяться коректно

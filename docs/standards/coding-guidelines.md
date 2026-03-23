# Coding Guidelines

Operational guide для щоденної розробки. Документ фіксує **як писати код**, не змінюючи зафіксовані архітектурні рішення.

> Canonical-рішення щодо архітектури, стану та API залишаються в `docs/architecture/*`. Якщо є розбіжність, пріоритет має architecture-документація.

---

## Scope

- Цей документ регулює стиль реалізації, структуру модулів, роботу з даними, тестування і типові антипатерни.
- Документ **не** змінює FSD-шари, auth flow, query policy, endpoint contracts.
- Для деталей дивись:
  - `docs/architecture/overview.md`
  - `docs/architecture/frontend-architecture.md`
  - `docs/architecture/state-management.md`
  - `docs/architecture/api-contracts.md`

---

## 1) General Coding Rules

### Rule: TypeScript-first, strict-safe code
**Why:** менше runtime-помилок, краща передбачуваність рефакторингів.

**Example**
```ts
// ✅
type ProjectId = string;

function getProjectName(project: { name: string }): string {
  return project.name;
}

// ❌
function getProjectName(project: any) {
  return project.name;
}
```

### Rule: Функції мають одну відповідальність
**Why:** легше тестувати, читати і перевикористовувати.

**Example**
```ts
// ✅ окремо трансформація
function mapProjectToOption(project: Project) {
  return { value: project.slug, label: project.name };
}
```

### Rule: Не використовувати "магічні" значення без контексту
**Why:** приховані бізнес-правила ламаються непомітно.

**Example**
```ts
// ✅
const DEFAULT_PAGE_SIZE = 20;
const offset = (page - 1) * DEFAULT_PAGE_SIZE;
```

### Rule: Імпорти тільки через публічний API slice
**Why:** ізоляція внутрішньої структури slice, стабільність імпортів.

**Example**
```ts
// ✅
import { useProjects } from '@/entities/project';

// ❌
import { useProjects } from '@/entities/project/api/use-projects';
```

---

## 2) React Component Guidelines

### Rule: UI-компонент не повинен знати transport-деталі
**Why:** розділення відповідальностей між UI і data access.

**Example**
```ts
// ✅ компонент споживає хук, а не axios напряму
const { data, isLoading, isError } = useProjects();
```

### Rule: Явно обробляти стани `loading`, `empty`, `error`, `success`
**Why:** передбачуваний UX і менше "німих" екранів.

**Example**
```tsx
if (isLoading) return <PageLoader />;
if (isError) return <ErrorState />;
if (!data?.length) return <EmptyState />;
return <ProjectsTable data={data} />;
```

### Rule: Контрольовані форми через `@mantine/form`
**Why:** консистентна валідація, dirty-tracking, керовані помилки полів.

### Rule: Базова доступність обов'язкова
**Why:** клавіатурна навігація і підтримка screen reader.

**Example**
- кнопки мають зрозумілий текст/aria-label
- інпути пов'язані з label
- destructive actions мають confirm-крок

---

## 3) State Management Rules

### Rule: Server state тільки через TanStack Query
**Why:** єдине джерело правди, керований кеш та інвалідація.

### Rule: Zustand лише для мінімального client state
**Why:** уникнення дублювання даних і розсинхронізації.

**Allowed у store**
- access token
- дрібний UI state, який не є server state і не підходить для URL

### Rule: Form state локально в компоненті
**Why:** форми не повинні засмічувати глобальний стан.

### Rule: Filter/pagination/view state у URL search params
**Why:** deep-linking, shareable URL, коректний back/forward flow.

### Rule: Підписка на Zustand тільки через selector
**Why:** мінімум зайвих re-render.

**Example**
```ts
// ✅
const token = useAuthStore((s) => s.accessToken);

// ❌
const auth = useAuthStore();
```

Деталі: `docs/architecture/state-management.md`.

---

## 4) API Interaction Rules

### Rule: axios викликається тільки у `entities/*/api` або `shared/api`
**Why:** єдина точка контролю auth, retries, errors.

### Rule: features працюють через entity API + mutation hooks
**Why:** features описують user action, а не транспорт.

### Rule: 401 flow не дублювати в компонентах
**Why:** refresh/retry вже централізовано у interceptor.

### Rule: Помилки форми мапити через `getFieldErrors`
**Why:** точне відображення серверної валідації біля відповідних полів.

Деталі: `docs/architecture/api-contracts.md`.

---

## 5) Query/Mutation Conventions

### Rule: Query keys тільки через key factory
**Why:** стабільна інвалідація і передбачувана структура кешу.

### Rule: Інвалідація точкова, а не широка
**Why:** широка інвалідація скидає зайві дані і погіршує UX.

**Example**
```ts
// ✅
queryClient.invalidateQueries({ queryKey: projectKeys.lists() });

// ❌
queryClient.invalidateQueries({ queryKey: ['projects'] });
```

### Rule: Оптимістичні оновлення лише там, де це явно дозволено
**Why:** зайва оптимістика ускладнює rollback і збільшує ризик багів.

### Rule: Пагінація UI (`page`) коректно конвертується в API (`offset`)
**Why:** уникнення зміщених результатів і "дублів" між сторінками.

Деталі: `docs/architecture/state-management.md`.

---

## 6) Styling and UI Rules

### Rule: Використовувати Mantine-підхід консистентно
**Why:** єдиний UX і швидша підтримка компонентів.

### Rule: Не використовувати inline styles для постійного UI
**Why:** складніше підтримувати, перевикористовувати та уніфікувати.

### Rule: Повторювані UI-рішення виносити у `shared/ui` або `widgets`
**Why:** менше дублювання і однакові патерни поведінки.

---

## 7) Testing Minimums

### Rule: Кожна суттєва feature має перевірку критичного сценарію
**Why:** захист основного user flow від регресій.

### Rule: Entities покриваються unit/integration тестами на API-hooks та маппери
**Why:** саме тут найбільший ризик контрактних помилок.

### Rule: Баг-фікс супроводжується тестом на відтворення
**Why:** гарантія, що дефект не повернеться.

Орієнтири покриття і стек тестів: `docs/architecture/frontend-architecture.md`.

---

## 8) Anti-Patterns Checklist

- дублювати server data у Zustand
- прямі імпорти з внутрішніх файлів slice замість `index.ts`
- axios-виклики з `features/ui/components`
- широка інвалідація кешу "по префіксу"
- локальна реалізація refresh token flow в окремих хуках
- ігнорування `loading/empty/error` станів
- використання `any` без вагомої причини

---

## Definition of Done for Code Changes

Перед завершенням задачі перевір:

- код відповідає правилам цього документа
- не порушені інваріанти з `docs/architecture/*`
- для нової поведінки додані або оновлені тести
- у diff немає архітектурних "обхідних шляхів" (direct imports, ad-hoc API layer)

# Coding Guidelines

Operational guide для щоденної розробки. Документ фіксує **як писати код**, не змінюючи зафіксовані архітектурні рішення.

> Canonical-рішення щодо архітектури, стану та API залишаються в `docs/architecture/*`. Якщо є розбіжність, пріоритет має architecture-документація.

---

## Scope

- Цей документ регулює стиль реалізації, структуру модулів, роботу з даними, тестування і типові антипатерни.
- Документ **не** змінює шари Page-First, auth flow, query policy, endpoint contracts.
- Для деталей дивись:
  - `docs/architecture/overview.md`
  - `docs/architecture/frontend-architecture.md`
  - `docs/architecture/state-management.md`
  - `docs/architecture/api-contracts.md`
- Принципи розподілу відповідальностей (GRASP, SOLID, орієнтири чистого коду) — [`grasp-solid.md`](./grasp-solid.md).
- Стиль під **React Compiler** без зайвої ручної мемоізації — [`react-compiler-readiness.md`](./react-compiler-readiness.md).
- Іменування файлів і стрілочні функції — [`naming-conventions.md`](./naming-conventions.md).

---

## 1) General Coding Rules

### Rule: TypeScript-first, strict-safe code

**Why:** менше runtime-помилок, краща передбачуваність рефакторингів.

**Example**

```ts
// ✅
type ProjectId = string;

const getProjectName = (project: { name: string }): string => project.name;

// ❌
const getProjectName = (project: any) => project.name;
```

### Rule: Функції мають одну відповідальність

**Why:** легше тестувати, читати і перевикористовувати.

**Example**

```ts
// ✅ окремо трансформація
const mapProjectToOption = (project: Project) => ({
  value: project.slug,
  label: project.name,
});
```

### Rule: Не використовувати "магічні" значення без контексту

**Why:** приховані бізнес-правила ламаються непомітно.

**Example**

```ts
// ✅
const DEFAULT_PAGE_SIZE = 20;
const offset = (page - 1) * DEFAULT_PAGE_SIZE;
```

### Rule: Імпорти через публічний API модуля (feature / shared)

**Why:** ізоляція внутрішньої структури модуля, стабільність імпортів.

**Example**

```ts
// ✅ feature з публічного barrel
import { CreateProjectModal } from "@/features/create-project";

// ✅ query-хук сторінки — з мікромодуля сторінки (підключення з роутера, не deep ззовні)
import { useProjects } from "@/pages/app/projects/queries/use-projects";

// ❌ deep-import у внутрішність чужої сторінки
import { something } from "@/pages/app/projects/internal-helper";
```

---

## 2) React Component Guidelines

Перед оптимізацією ре-рендерів див. [`react-compiler-readiness.md`](./react-compiler-readiness.md). Розподіл логіки по модулях — [`grasp-solid.md`](./grasp-solid.md).

### Rule: UI-компонент не повинен знати transport-деталі

**Why:** розділення відповідальностей між UI і data access.

**Example**

```ts
// ✅ компонент споживає хук Query, а не мережевий клієнт напряму
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

### Rule: Клієнтський стан і сесія — Context і локальний стан

**Why:** узгоджено з каноном: access token **in-memory** у провайдері + синхронний «міст» для `shared/api` (див. [State Management](../architecture/state-management.md#клієнтський-стан-client-state)).

**Zustand** у проєкті **не** є типовим вибором; якщо колись з’явиться — лише після явного архітектурного рішення (як у [Overview](../architecture/overview.md)).

### Rule: Form state локально в компоненті

**Why:** форми не повинні засмічувати глобальний стан.

### Rule: Filter/pagination/view state у URL search params

**Why:** deep-linking, shareable URL, коректний back/forward flow.

Деталі: [State Management](../architecture/state-management.md).

---

## 4) API Interaction Rules

### Rule: мережеві виклики лише в `shared/api`

**Why:** єдина точка контролю auth, retries, errors; обгортка над `fetch` / `ofetch` (див. [API Contracts](../architecture/api-contracts.md)).

### Rule: `pages/*` і `features/*` споживають API через Query-хуки та функції з `shared/api`

**Why:** UI і сценарії не дублюють транспорт; `queryFn` / `mutationFn` викликають типізовані функції з `shared/api`.

### Rule: 401 flow не дублювати в компонентах

**Why:** refresh/retry уже централізовано в HTTP-обгортці `shared/api`.

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
queryClient.invalidateQueries({ queryKey: ["projects"] });
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

### Rule: Повторювані UI-рішення виносити у `shared/ui` або в `features/*` / локальні `pages/.../ui`

**Why:** менше дублювання; окремого шару `widgets/` у проєкті немає (див. [Folder Structure](./folder-structure.md)).

---

## 7) Testing Minimums

### Rule: Кожна суттєва feature має перевірку критичного сценарію

**Why:** захист основного user flow від регресій.

### Rule: Критичні `shared/api`, query-хуки та маппери покривати unit/integration тестами

**Why:** саме тут найбільший ризик контрактних помилок.

### Rule: Баг-фікс супроводжується тестом на відтворення

**Why:** гарантія, що дефект не повернеться.

Орієнтири покриття і стек тестів: `docs/architecture/frontend-architecture.md`.

---

## 8) Anti-Patterns Checklist

- дублювати server data у глобальному клієнтському store замість TanStack Query
- прямі імпорти з внутрішніх файлів модуля замість публічного API (`index.ts` feature тощо)
- «сирий» HTTP / обхід `shared/api` з компонентів або з `pages/.../ui`
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

# Naming Conventions

Operational standard для єдиного іменування в кодовій базі. Документ визначає назви файлів, папок, символів, API-артефактів і тестів.

> Canonical-рішення щодо архітектури й меж шарів залишаються в `docs/architecture/*`. Якщо є розбіжність, пріоритет має architecture-документація.

---

## Scope

- Документ регулює **іменування**, а не структуру шарів чи state-стратегію.
- Мета: зменшити неоднозначність і прискорити code review.
- Пов'язані документи:
  - `docs/architecture/frontend-architecture.md`
  - `docs/standards/coding-guidelines.md`
  - `docs/standards/folder-structure.md`

---

## 1) File System Naming

### Rule: Папки slice-ів і сегментів у `kebab-case`
**Why:** однакова читабельність у файловій системі, передбачуваний пошук.

**Example**
```text
✅ features/translation-key/
✅ features/create-project/
✅ pages/app/projects/
❌ features/CreateProject/
❌ features/translation_key/
```

### Rule: Імена файлів — **малі літери**, слова **через дефіс** (`kebab-case`)

**Why:** передбачуваний вигляд на диску, краща читабельність довгих назв, один стиль для всього репозиторію.

**Pattern**
- тільки `a-z`, `0-9`, `-`; без пробілів і без `camelCase`/`PascalCase`/`snake_case` **у назві файлу**.
- розширення як завжди: `.ts`, `.tsx`, `.css` тощо.

**Example**
```text
✅ use-projects.ts
✅ project-api.ts
✅ create-project-modal.tsx
✅ product-type.ts
❌ useProjects.ts
❌ ProjectAPI.ts
❌ Product_Type.ts
```

### Rule: Не розділяти «логічні частини» назви файлу **крапкою**

**Why:** крапка зарезервована під розширення; `product.type.ts` плутає з подвійним розширенням і гірше сортується в списках.

**Example**
```text
✅ product-type.ts
✅ api-error.ts
❌ product.type.ts
❌ user.dto.ts
```

Якщо треба кількість типів у назві — використовуй дефіси: `project-create-request.ts`, `paginated-response.ts` (або один доменний файл `project-types.ts`).

### Rule: Публічний API модуля завжди `index.ts`
**Why:** одна точка імпорту ззовні (наприклад `features/*/index.ts`).

---

## 2) React and TypeScript Symbols

### Rule: Функції оголошувати **стрілочними** (`const … = () => {}`), не ключовим словом `function`

**Why:** єдиний стиль локальних і експортованих функцій, зручніше для колбеків і узгоджено з командною домовленістю; не змішувати `function foo()` і `const foo = () => {}` без потреби.

**Example**
```ts
export const formatSlug = (value: string) => value.trim().toLowerCase();

export const useProjects = () => {
  // ...
};
```

Винятки узгоджуються в review (наприклад, генератори з `function*`, або вимога лінтера до hoisting — рідко).

### Rule: React компоненти — ім’я експорту в `PascalCase`, реалізація стрілочною
**Why:** стандарт React ecosystem і чітке відрізнення від HTML-тегів.

**Example**
```tsx
export const TranslationTable = () => {
  return null;
};
```

### Rule: Hooks — `camelCase` + префікс `use`
**Why:** миттєво видно, що функція підпадає під hook rules.

**Example**
```ts
export const useProjects = () => {};
export const useCreateProject = () => {};
```

### Rule: Утиліти/хелпери — `camelCase` дієслівно-предметні назви
**Why:** назва пояснює дію, а не місце зберігання.

**Example**
```ts
formatDate()
buildQueryParams()
getFieldErrors()
```

### Rule: Типи/інтерфейси — `PascalCase`
**Why:** візуально відділяються від змінних і функцій.

**Example**
```ts
type Project = {};
interface ApiError {}
```

### Rule: Константи
**Why:** різні рівні констант мають різні правила.

**Pattern**
- module-level/глобальні константи: `UPPER_SNAKE_CASE` (`DEFAULT_PAGE_SIZE`)
- локальні значення в межах функції: `camelCase` (`pageSize`)

---

## 3) API Naming

### Rule: API-функції іменуються дієсловом + сутністю
**Why:** назва одразу відображає HTTP intent і бізнес-об'єкт.

**Example**
```ts
fetchProjects()
fetchProject()
createProject()
updateProject()
deleteProject()
```

### Rule: Файли API-слою називати за сутністю або дією
**Why:** легкий пошук за доменом.

**Example**
```text
project-api.ts
use-projects.ts
use-create-project.ts
```

### Rule: Request/Response типи з предметним суфіксом
**Why:** уникнення конфліктів між близькими сутностями.

**Example**
```ts
type ProjectCreateRequest = {};
type ProjectResponse = {};
type PaginatedResponse<T> = {};
```

---

## 4) State and Query Naming

### Rule: Query key factory — `<entity>Keys`
**Why:** уніфікований API для інвалідації і читання кешу.

**Example**
```ts
projectKeys.all
projectKeys.lists()
projectKeys.detail(slug)
```

### Rule: Mutation hooks — `use<Action><Entity>`
**Why:** читабельний контракт user action.

**Example**
```ts
useCreateProject()
useUpdateTranslation()
useBulkDeleteKeys()
```

### Rule: Клієнтський store (якщо використовується) — `use<Domain>Store` або контекст `use<Domain>`
**Why:** явне позначення джерела client state. Деталі шару стану — `docs/architecture/state-management.md`.

**Example**
```ts
useAuthStore // лише якщо обрано Zustand після явного рішення
// або: useAuth() з React Context
```

### Rule: Селектори / accessors називати за значенням
**Why:** зрозумілий намір і мінімізація зайвих re-render.

**Example**
```ts
const accessToken = useAuth((s) => s.accessToken);
```

---

## 5) UI and i18n Naming

### Rule: UI-компоненти називати за роллю, не за візуальним випадком
**Why:** компонент може переїхати в інший layout без перейменування.

**Example**
```text
project-list-toolbar.tsx   ✅
top-blue-panel.tsx         ❌
```

### Rule: Variant-суфікси для одного сімейства UI
**Why:** легка навігація між пов'язаними компонентами.

**Example**
```text
translation-table.tsx
translation-table-row.tsx
translation-table-empty-state.tsx
```

### Rule: i18n keys у `snake_case` з domain-префіксом
**Why:** стабільні ключі без залежності від конкретного UI-тексту.

**Example**
```text
project.create_success
auth.login_invalid_credentials
translations.bulk_delete_confirm
```

---

## 6) Test Naming

### Rule: Unit/Integration тести — `*.test.ts` / `*.test.tsx`
**Why:** стандартний патерн для runner/tooling.

**Example**
```text
use-projects.ts
use-projects.test.ts
```

### Rule: Назви `describe` і `it` відображають бізнес-поведінку
**Why:** тест читається як специфікація.

**Example**
```ts
describe('useCreateProject', () => {
  it('invalidates project list after successful creation', () => {});
});
```

---

## 7) Anti-Patterns

- **у назвах файлів** `camelCase` / великі літери / підкреслення замість дефіса
- **крапка всередині** базового імені файлу: `foo.type.ts`, `bar.dto.ts` (замість `foo-type.ts` / доменного `*-types.ts`)
- змішувати стилі імен папок без правила
- оголошувати звичайні функції через `function name()` там, де достатньо стрілочної форми (див. правило вище)
- короткі абревіатури без контексту: `trKey`, `cfg`, `tmp`
- нечіткі назви: `helper.ts`, `data.ts`, `common.ts`
- дублювати префікс у назві: `project-project-api.ts`
- називати компонент за стилем, а не за роллю (`green-button.tsx`)
- називати hooks без `use` префікса

---

## 8) Naming Review Checklist

Перед merge перевір:

- **файли:** малі літери, слова через `-`, без крапок у базовому імені; папки в `kebab-case`
- **код:** компоненти/типи в `PascalCase`; hooks/утиліти стрілочні, `camelCase`
- API-функції мають дієслівно-предметні назви (`fetch/create/update/delete`)
- query key factory має формат `<entity>Keys`
- i18n ключі в `snake_case` і з доменним префіксом
- у diff немає загальних неінформативних назв (`utils`, `temp`, `stuff`)

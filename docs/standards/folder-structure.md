# Folder Structure

Операційний стандарт для розміщення файлів у `src/`. Документ визначає **де саме створювати код** та як уникати хаотичної структури.

> **Canonical-джерело:** [`docs/architecture/overview.md`](../architecture/overview.md) та [`docs/architecture/frontend-architecture.md`](../architecture/frontend-architecture.md). Якщо виникає конфлікт, пріоритет має architecture-документація.

---

## Scope

- Цей документ описує правила розміщення файлів і папок для архітектури **Page-First**.
- Документ **не** змінює імпортні межі, auth flow та query policy — вони в `docs/architecture/*`.
- Пов’язані джерела:
  - `docs/architecture/overview.md`
  - `docs/architecture/frontend-architecture.md`
  - `docs/architecture/state-management.md`
  - `docs/standards/coding-guidelines.md`
  - `docs/standards/naming-conventions.md`

---

## 1) Базове дерево `src/`

```text
src/
├── app/
├── pages/
├── features/
└── shared/
```

Окремих top-level шарів **`widgets/`** та **`entities/`** немає: великі блоки збираються на **сторінці** або виносяться в **`features/*`**; доменні типи й HTTP можуть жити в **`shared/api`** та поруч із сценаріями в **`pages/*`** / **`features/*`**.

### Rule: Не створювати нові top-level папки поруч із цими чотирма шарами

**Why:** єдина точка навігації й передбачуваність для людей і агентів.

**Example**

```text
✅ src/features/create-project/
✅ src/pages/app/projects/
❌ src/modules/create-project/
❌ src/widgets/dashboard/
```

---

## 2) Шар `app/`

Bootstrap і інфраструктура маршрутизації **без** бізнес-сценаріїв продукту.

**Рекомендована структура**

```text
app/
├── providers/          # QueryClient, Mantine, Router, Auth, i18n
├── routes/             # або routing/ — декларація маршрутів TanStack Router
├── layouts/          # public / authenticated shell, _authenticated
├── styles/
└── index.tsx
```

Деталі: [Frontend Architecture](../architecture/frontend-architecture.md) (розділ «Шар app/»).

---

## 3) Шар `shared/`

Перевикористовуваний код **без** залежності від конкретних сторінок і фіч. Сегменти всередині `shared/` можуть імпортувати один одного; **`shared/` не імпортує** `pages/`, `features/`, `app/`.

**Рекомендована структура**

```text
shared/
├── api/           # fetch/ofetch-обгортка, доменні *-api.ts, query-client.ts
├── query-keys/    # усі TanStack Query key factory (*.ts)
├── auth/          # AuthProvider, синхронний доступ для HTTP-шару
├── config/
├── i18n/
├── ui/            # глобальні примітиви, обгортки над UI-бібліотекою
├── lib/
└── test/          # MSW handlers, тестові утиліти
```

Деталі: [Architecture Overview](../architecture/overview.md) (шари Page-First).

---

## 4) Шар `pages/`

Кожна **сторінка** (маршрут або логічний екран) — **окремий мікромодуль**. Сусідні сторінки **не імпортують** одна одну.

**Типова структура мікромодуля**

```text
pages/app/projects/
├── projects-page.tsx    # точка входу (композиція)
├── components/
├── hooks/
├── queries/             # useQuery/useMutation лише для цього екрана
├── mappers/
├── types/
└── ui/
```

Додавай лише ті підпапки, які реально використовуються. Імена файлів — [naming-conventions](./naming-conventions.md) (`kebab-case`, малі літери).

**Query keys** у коді не тримаємо локально в сторінці — імпорт з **`shared/query-keys/`**.

---

## 5) Шар `features/`

Перевикористання **між сторінками**. Одна `feature` **не** імпортує іншу `feature`; композиція — на **`pages/*`** (props / callbacks). Див. [Overview](../architecture/overview.md) (композиція двох features на сторінці).

**Типова структура**

```text
features/create-project/
├── create-project-modal.tsx
├── use-create-project.ts
├── types.ts              # за потреби
└── index.ts              # публічний API feature (бажано)
```

---

## 6) Placement Rules by Artifact

### Rule: HTTP і query client

**Placement**

- **`shared/api/*`** — клієнт мережі, refresh/retry, типізовані функції запитів.
- **`pages/.../queries/*`** — хуки `useQuery` / оркестрація даних **екрана**.
- **`features/.../*`** — спільні між сторінками хуки запитів/мутацій (наприклад `use-create-project.ts`).

### Rule: Query key factory

**Placement**

- **`shared/query-keys/*.ts`** — єдине місце для key factory; імпорт лише звідси.

### Rule: Типи та маппери

**Placement**

- типи та маппери DTO → UI поруч із споживачем: `pages/.../types`, `pages/.../mappers` або `features/...`;
- справді глобальні типи транспорту — поруч із **`shared/api`** (узгоджено з командою).

### Rule: UI

**Placement**

- **`pages/.../ui/`**, **`pages/.../components/`** — презентація цього екрана;
- **`features/...`** — UI блоку, який підключають кілька сторінок;
- **`shared/ui/`** — примітиви без продуктової семантики.

Великий блок сторінки без перевикористання **не** виносимо в окремий шар `widgets/` — лишаємо в `pages/` або дробимо на локальні `components/`.

### Rule: Утиліти

**Placement**

- **`shared/lib/*`** — загальні pure helpers без домену;
- **`pages/.../hooks`**, **`features/...`** — вузькі утіліти під сценарій (не копіювати в третій місце — тоді в `shared/lib`).

---

## 7) Публічний API (`index.ts`)

### Rule: `features/*` бажано експортувати через `index.ts`

**Why:** стабільний імпорт `@/features/create-project` без deep paths.

### Rule: `pages/*` — `index.ts` за потреби

**Why:** сторінка зазвичай підключається з роутера напряму до `*-page.tsx`; додатковий barrel не обов’язковий.

### Rule: Уникати deep-importів з чужого модуля

**Why:** межа модуля — сторінка або feature; зовні імпортують публічний контракт.

**Example**

```ts
// ✅
import { CreateProjectModal } from '@/features/create-project';

// ❌ (чужа внутрішність сторінки)
import { Something } from '@/pages/app/projects/internal-helper';
```

---

## 8) Тести

**Placement**

- `*.test.ts` / `*.test.tsx` **поруч** із файлом, який тестується (co-location).

**Example**

```text
pages/app/projects/queries/use-projects.ts
pages/app/projects/queries/use-projects.test.ts
```

- **`shared/test/*`** — MSW handlers і спільна тест-інфраструктура.

---

## 9) Імпортні межі

**Напрямок залежностей:** `app` → `pages` → `features` → `shared`.

- **`shared/*`** не залежить від `pages`, `features`, `app`.
- **`pages/*`** не імпортує інші **`pages/*`**.
- **`features/*`** не імпортує інші **`features/*`**.

Деталі: [Frontend Architecture](../architecture/frontend-architecture.md) (розділ «Правила імпортів (Page-First)»).

---

## 10) Anti-Patterns Checklist

- нові top-level папки поза `app` / `pages` / `features` / `shared`
- **`feature` → `feature`** прямий імпорт (замість композиції на сторінці)
- **`pages` → `pages`** імпорт сусіднього маршруту
- query key factory в `pages/` або `features/` замість **`shared/query-keys/`**
- «сірий» HTTP з компонента замість **`shared/api`** + Query
- дублювання однакових примітивів замість **`shared/ui`**
- порожні папки `queries/`, `components/` «про запас»
- вся логіка екрана в одному гігантському файлі замість мікромодуля сторінки

---

## 11) Structure Review Checklist

Перед merge перевір:

- нові файли лежать у правильному шарі Page-First (`app` / `pages` / `features` / `shared`)
- для нової **`feature`** є **`index.ts`**, якщо зовні її імпортують
- ключі TanStack Query лише з **`shared/query-keys/`**
- немає заборонених горизонтальних імпортів (`pages↔pages`, `features↔features`)
- тести co-located або спільна інфра в **`shared/test/*`**

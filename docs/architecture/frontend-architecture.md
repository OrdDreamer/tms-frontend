# Frontend Architecture

Детальний опис структури проєкту, патернів та конфігурацій. Читати перед будь-якою роботою з кодом.

> **Для агентів:** дотримуватись **Page-First** шарів (`app` → `pages` → `features` → `shared`). Не змінювати структуру шарів без явного дозволу. Узгодженість з [Architecture Overview](./overview.md).

---

## Routing: Структура сторінок

```
/                           → лендинг / маркетингова сторінка
/roadmap                    → плани розвитку продукту
/updates                    → оновлення та новини

/auth/login                 → логін
/auth/forgot-password       → запит скидання пароля
/auth/reset-password/:token → скидання пароля (токен з email)

/app                        → дашборд (захищена зона)
/app/projects               → список проєктів
/app/projects/:projectSlug    → дашборд проєкту

/app/projects/:projectSlug/translations   → робота з перекладами
/app/projects/:projectSlug/settings       → налаштування проєкту
/app/projects/:projectSlug/data/import    → імпорт перекладів
/app/projects/:projectSlug/data/export    → експорт перекладів

/app/profile                → профіль та налаштування користувача
```

**Auth guard:** реалізований через `beforeLoad` на layout-маршруті `_authenticated` (TanStack Router). Всі маршрути під `/app/*` захищені цим layout-маршрутом.

**Query parameters для `/translations`:** `search`, `lang`, `status` (translated/untranslated/all), `sort`, `page`, `limit`, `view` (table/list).

**Code splitting:** вимкнено — всі сторінки в одному бандлі.

---

## Page-First: шари

Чотири шари від найвищого рівня абстракції до найнижчого. Залежності — **тільки зверху вниз** ([детальні правила](./overview.md#шари-page-first)).

```
src/
├── app/        # Точка входу, провайдери, маршрути, layouts, глобальні стилі
├── pages/      # Сторінки: кожна — мікромодуль (queries, components, …)
├── features/   # Перевикористання між сторінками, трохи бізнес-логіки
└── shared/     # Інфраструктура: api, query-keys, ui-примітиви, i18n, util
```

`widgets/` та `entities/` **не виділяються** окремими шарами: великі блоки збираються на сторінці або у `features/`, доменні типи та HTTP можуть жити в `shared/api` і поруч із use case у `pages/` / `features/`.

> Актуальний список файлів — у файловій системі. Цей документ фіксує **патерни**, а не інвентар.

---

## Модулі сторінок та features

### Сторінка (`pages/<area>/<page>/`)

Кожен маршрут (або логічний екран) — **окремий мікромодуль**. Сусідні сторінки **не імпортують** одна одну.

Типова структура (сегменти додавати за потреби):

```
pages/app/projects/
├── projects-page.tsx       # точка входу сторінки (композиція)
├── components/
├── hooks/
├── queries/                # useQuery / useMutation, оркестрація лише для цього екрана
├── mappers/
├── types/
└── ui/                     # презентаційні шматки саме цього екрана
```

### Feature (`features/<name>/`)

Перевикористовуваний блок між сторінками. **Не імпортує** інші `features/*` — якщо потрібна звʼязка, композиція на рівні `pages/*` (props / callbacks), див. [Overview](./overview.md#як-працювати-коли-одна-feature-сильно-залежить-від-іншої).

Типово:

```
features/create-project/
├── create-project-modal.tsx
├── use-create-project.ts   # useMutation + invalidate через ключі з shared/query-keys
├── types.ts                # за потреби
└── index.ts                # публічний API feature (бажано)
```

### Сегменти: зведена таблиця

| Сегмент      | pages | features |
|-------------|:-----:|:--------:|
| `components/`, `ui/` | ✅ | ✅ |
| `hooks/` | ✅ | ✅ |
| `queries/` або `*-queries.ts` | ✅ | ✅ |
| `mappers/`, `types/` | ✅ | за потреби |
| `index.ts` (public API) | за потреби | ✅ бажано |

---

## Приклади

**Сторінка списку проєктів (`pages/app/projects/`):**

```
projects/
├── projects-page.tsx
├── queries/
│   └── use-projects.ts       # useQuery; queryKey з shared/query-keys/project-keys
└── types.ts                  # за потреби
```

Чисті виклики API — у `shared/api/` (`fetchProjects()` тощо). **Key factory** — лише в [`shared/query-keys/`](./state-management.md#query-key-factory).

**Feature створення проєкту (`features/create-project/`):**

```
create-project/
├── create-project-modal.tsx
├── use-create-project.ts     # useMutation; invalidateQueries(projectKeys.lists())
└── index.ts
```

---

## Шар app/

`app/` — без «слайсів», лише інфраструктура запуску:

```
app/
├── providers/          # MantineProvider, QueryClientProvider, RouterProvider, AuthProvider, …
├── routes/ або routing/ # TanStack Router: оголошення маршрутів, `beforeLoad`, звʼязок path → page
├── layouts/            # Public / authenticated shell, `_authenticated` layout
├── styles/             # global.css
└── index.tsx           # root: провайдери + роутер
```

---

## Шар shared/

`shared/` містить сегменти напряму (без slice-ів). Стандартні та кастомні:

```
shared/
├── api/         # обгортка fetch/ofetch, refresh/retry, query-client.ts, доменні *-api.ts
├── query-keys/  # усі TanStack Query key factory (*.ts)
├── auth/        # AuthProvider, синхронний доступ до токена для HTTP (див. state-management)
├── config/      # env.ts — типізовані змінні середовища
├── i18n/        # config.ts + locales/en/, locales/uk/
├── ui/          # UI-примітиви без бізнес-логіки (error-boundary, page-loader)
├── lib/         # утиліти загального призначення
└── test/msw/    # MSW handlers — unit / integration / dev
```

> У межах `shared/` сегменти можуть імпортувати один одного (наприклад `shared/api` + `shared/auth` для заголовка `Authorization` та refresh). **`shared/` не залежить від `pages/`, `features/`, `app/`.**

---

## Правила імпортів (Page-First)

Дозволений напрямок: **`app` → `pages` → `features` → `shared`**.

```
✅ pages/app/projects може імпортувати з features/* та shared/*
✅ features/create-project може імпортувати з shared/* (api, query-keys, ui, …)
✅ app/* може імпортувати з pages, features, shared (композиція маршрутів)
❌ shared/* НЕ імпортує pages, features, app
❌ pages/* НЕ імпортує інші pages/* (сусідні маршрути — окремі модулі)
❌ features/* НЕ імпортує інші features/* (композиція на сторінці або спільне в shared)
```

**Виняток:** сегменти всередині `app/` та `shared/` можуть імпортувати один одного (див. блок `shared/` вище).

**Спільні дані без крос-імпортів фіч:** типи та HTTP — `shared/api`; **одна форма query keys** — `shared/query-keys`.

Правила enforced через `eslint-plugin-boundaries` (конфіг у `eslint.config.js`).

Для `features/*` бажано **публічний API** через `index.ts`. Для `pages/*` допустимі прямі імпорти всередині мікромодуля сторінки; зовнішні споживачі підключаються через маршрут, а не через deep import іншої сторінки.

```ts
// ✅ з feature — через публічний API
import { CreateProjectModal } from '@/features/create-project'

// ✅ ключі кешу — завжди з shared
import { projectKeys } from '@/shared/query-keys/project-keys'

// ❌ feature A не імпортує feature B
```

---

## Конвенції іменування

| Артефакт | Конвенція | Приклад |
|----------|-----------|---------|
| Папки (features / групи сторінок) | kebab-case | `create-project/`, `app/projects/` |
| Всі файли | kebab-case | `translation-table.tsx`, `use-projects.ts`, `project-api.ts` |
| React компоненти (назва) | PascalCase | `export function TranslationTable` |
| Хуки (назва) | camelCase з префіксом `use` | `export function useProjects` |
| Утиліти / helpers (назва) | camelCase | `export function formatDate` |
| TypeScript типи/інтерфейси | PascalCase | `type Project`, `interface ApiError` |
| Query keys об'єкти | camelCase | `projectKeys.all`, `projectKeys.detail(id)` |
| API функції | camelCase | `fetchProjects()`, `createProject()` |
| i18n ключі | snake_case | `project.create_success` |

---

## Dev-середовище та проксі

**Локально (Vite dev server):**
```ts
// vite.config.ts
server: {
  proxy: {
    '/api': {
      target: process.env.VITE_API_BASE_URL || 'http://localhost:8000',
      changeOrigin: true,
    }
  }
}
```

`VITE_API_BASE_URL` у `.env.local` визначає куди проксювати — локальний бекенд або dev-сервер. Так вирішується CORS у dev.

**Dev-сервер / Production:**
Nginx проксює `/api/*` → бекенд. Фронтенд робить запити на той самий домен — без CORS.

```nginx
location /api/ {
    proxy_pass http://backend:8000/api/;
}

location / {
    try_files $uri $uri/ /index.html;  # SPA fallback
}
```

---

## Error Handling

### React: Error Boundary (`app/`)

**Глобальний `<ErrorBoundary>`** (або еквівалент з Mantine / `react-error-boundary`) розміщується **навколо дерева маршрутів / основного layout** у `app/` — щоб при необробленій помилці в дочірньому дереві показати fallback-екран (і за потреби лог / Sentry), а не білий екран.

**Ловить:**
- помилки під час **рендеру** React-компонентів;
- помилки в **lifecycle** клас-компонентів (якщо є);
- помилки в **конструкторах** дочірніх компонентів у межах дерева під boundary.

**Не ловить** (треба окремий try/catch, `onError` у Query, обробники подій):
- помилки в **обробниках подій** (`onClick`, `onSubmit` тощо);
- **асинхронний** код поза рендером (`setTimeout`, `.then()` без пробросу, частина коду в `useEffect` після await);
- помилки **самого** boundary;
- помилки **SSR** (для SPA неактуально).

Тобто boundary **не замінює** обробку помилок API — вона паралельна: рантайм UI ламається → boundary; мережа / 4xx / 5xx → Query / `handleGlobalError` / локальні стани.

Детальніше про стани екрана `error` для даних — [UI Patterns](../standards/ui-patterns.md).

### API та мережа

**API-помилки** обробляються на двох рівнях:
- Глобально — через `QueryCache` / `MutationCache` callbacks у `shared/api/query-client.ts` (`handleGlobalError`, toast-и; див. [правила кодів](./state-management.md))
- Індивідуально — `onError` у запитах/мутаціях для специфічної UI-логіки (поля форми, `suppressGlobalError`)
- `401` — у `shared/api` (refresh + retry); TanStack Query бачить вже результат після повтору або фінальну помилку

Конфігурація QueryClient, auth flow, `BroadcastChannel` — у [State Management](./state-management.md).

---

## TypeScript конфігурація

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "target": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

## Тестування

| Рівень | Інструмент | Що покривається |
|--------|-----------|-----------------|
| Unit | Vitest | хелпери, `shared/api`, auth/session, query key factory (pure), hooks |
| Integration | Vitest + RTL + MSW | форми, компоненти з API-викликами |
| E2E | Playwright | login flow, CRUD проєктів, workflow перекладача |

Розміщення тестів: `*.test.ts(x)` поряд з source файлами (co-location).

MSW handlers: `shared/test/msw/handlers/` — перевикористовуються у unit, integration тестах та dev-режимі.

**Coverage gate (CI):** `70%` — hard threshold, CI fails нижче. Ціль для `features/` та критичних `pages/`: `80%`.

---

## Детальна документація

- [State Management](./state-management.md) — TanStack Query, client state, auth flow, query keys
- [API Contracts](./api-contracts.md) — endpoints, формати відповідей, коди помилок
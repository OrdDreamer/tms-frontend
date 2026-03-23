# Frontend Architecture

Детальний опис структури проєкту, патернів та конфігурацій. Читати перед будь-якою роботою з кодом.

> **Для агентів:** не створювати файли поза FSD-структурою. Не змінювати структуру шарів без явного дозволу. Всі архітектурні рішення зафіксовані тут.

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
/app/projects/:projectId    → дашборд проєкту

/app/projects/:projectId/translations   → робота з перекладами
/app/projects/:projectId/settings       → налаштування проєкту
/app/projects/:projectId/data/import    → імпорт перекладів
/app/projects/:projectId/data/export    → експорт перекладів

/app/profile                → профіль та налаштування користувача
```

**Auth guard:** реалізований через `beforeLoad` на layout-маршруті `_authenticated` (TanStack Router). Всі маршрути під `/app/*` захищені цим layout-маршрутом.

**Query parameters для `/translations`:** `search`, `lang`, `status` (translated/untranslated/all), `sort`, `page`, `limit`, `view` (table/list).

**Code splitting:** вимкнено — всі сторінки в одному бандлі.

---

## FSD: Шари

Шість шарів від найвищого до найнижчого рівня відповідальності. Залежності — тільки зверху вниз.

```
src/
├── app/        # Все що запускає додаток: providers, routing, global styles, entry point
├── pages/      # Повні сторінки або великі частини сторінок (nested routing)
├── widgets/    # Великі самодостатні блоки UI, що компонують features та entities
├── features/   # Перевикористовувані реалізації дій користувача з бізнес-логікою
├── entities/   # Бізнес-сутності, з якими працює проєкт (project, translation-key, user)
└── shared/     # Перевикористовуваний функціонал, відʼєднаний від бізнес-специфіки
```

`app/` і `shared/` — **не мають slice-ів**, містять сегменти напряму. Решта шарів (pages, widgets, features, entities) — складаються зі slice-ів.

> Актуальний список файлів — у файловій системі. Цей документ фіксує **патерни**, а не інвентар.

---

## Slice-и та сегменти

Slice — група коду, обʼєднана за бізнес-доменом. Кожен slice має:
- **Публічний API** — `index.ts`, єдина точка імпорту ззовні
- **Нуль звʼязків** з іншими slice-ами того ж шару (zero coupling)
- **Високу згуртованість** — весь код, повʼязаний з цією метою, живе разом

Споріднені slice-и можна групувати в папку (наприклад `features/auth/login/`, `features/auth/logout/`), але код між ними спільним бути не може.

### Стандартні сегменти (FSD)

```
<slice-name>/
├── ui/        # UI-компоненти, форматери, стилі
├── api/       # Взаємодія з бекендом: запити, TanStack Query хуки, маппери
├── model/     # Модель даних: типи, схеми, стори, бізнес-логіка
├── lib/       # Допоміжний код, що обслуговує інші модулі slice
├── config/    # Конфігурація, feature flags
└── index.ts   # Публічний API slice
```

> Назви сегментів описують **призначення**, а не сутність. Тому `api/` замість `hooks/`, `model/` замість `types/`.

Не кожен сегмент потрібен у кожному шарі. Додавай лише коли реально потрібен — порожні папки не створювати:

| Сегмент | pages | widgets | features | entities |
|---------|:-----:|:-------:|:--------:|:--------:|
| `ui/`   | ✅ | ✅ | ✅ | ✅ |
| `api/`  | — | — | ✅ | ✅ |
| `model/`| — | — | ✅ | ✅ |
| `lib/`  | — | — | за потреби | за потреби |
| `config/`| — | — | за потреби | — |

---

## Приклади slice-ів

**Entity (entities/project/):**
```
project/
├── api/
│   ├── project-api.ts       # fetchProjects(), createProject(), ...
│   ├── use-projects.ts      # useQuery — список проєктів
│   └── use-project.ts       # useQuery — один проєкт
├── model/
│   ├── types.ts             # type Project, ProjectCreate, ...
│   └── query-keys.ts        # projectKeys.all, projectKeys.detail(id)
└── index.ts
```

TanStack Query хуки (`useQuery`, `useMutation`) — це **взаємодія з бекендом**, тому живуть в `api/`, а не в окремому сегменті.

**Feature (features/create-project/):**
```
create-project/
├── ui/create-project-modal.tsx   # Mantine Modal з формою
├── api/use-create-project.ts     # useMutation + інвалідація кешу
└── index.ts
```

---

## Шар app/

`app/` містить сегменти напряму (без slice-ів):

```
app/
├── providers/     # MantineProvider, QueryClientProvider, RouterProvider, I18nProvider
├── styles/        # global.css
└── index.tsx      # Root компонент — монтує всі провайдери
```

---

## Шар shared/

`shared/` містить сегменти напряму (без slice-ів). Стандартні та кастомні:

```
shared/
├── api/        # http-client.ts (axios instance з interceptors), query-client.ts
├── auth/       # auth-store.ts — Zustand store (accessToken, user, clearAuth)
├── config/     # env.ts — типізовані змінні середовища
├── i18n/       # config.ts + locales/en/, locales/uk/
├── ui/         # UI-примітиви без бізнес-логіки (error-boundary, page-loader)
├── lib/        # Утиліти загального призначення (дати, валідація, форматування)
└── test/msw/   # MSW handlers — спільні для тестів і dev-режиму
```

> `auth/` — кастомний сегмент. `shared/api/http-client.ts` імпортує з `shared/auth/` для interceptor-а — це дозволено, бо в `shared/` сегменти можуть вільно імпортувати один з одного.

---

## Правила імпортів (FSD)

Дозволений напрямок: `pages` → `widgets` → `features` → `entities` → `shared`

```
✅ entities/project може імпортувати з shared/api
✅ features/create-project може імпортувати з entities/project
✅ pages/projects може імпортувати з widgets/ та features/
❌ entities НЕ може імпортувати з features або pages
❌ shared НЕ може імпортувати з будь-якого іншого шару
❌ Горизонтальні імпорти між slice-ами одного шару — за замовчуванням заборонені
```

**Виняток `app/` та `shared/`:** сегменти всередині цих шарів можуть вільно імпортувати один з одного (вони не мають slice-ів).

**Cross-entity references (`@x`):** за замовчуванням slice-и одного шару не можуть імпортувати один з одного. Але якщо entities семантично повʼязані (наприклад `translation-key` використовує тип з `project`), FSD дозволяє явний cross-import через публічний API (`index.ts`). Такі звʼязки мають бути мінімальними і задокументованими.

Правила enforced через `eslint-plugin-boundaries` (конфіг у `eslint.config.js`).

Кожен slice експортує свій публічний API виключно через `index.ts`. Прямі імпорти з внутрішніх файлів slice — заборонені.

```ts
// ✅ правильно
import { useProjects } from '@/entities/project'

// ❌ заборонено — прямий імпорт з внутрішнього файлу slice
import { useProjects } from '@/entities/project/api/use-projects'
```

---

## Конвенції іменування

| Артефакт | Конвенція | Приклад |
|----------|-----------|---------|
| Папки (FSD slices) | kebab-case | `translation-key/`, `create-project/` |
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

**Глобальний `<ErrorBoundary>`** на рівні `app/` — ловить всі React-помилки рендеру.

**API-помилки** обробляються на двох рівнях:
- Глобально — через `QueryCache` / `MutationCache` callbacks у `shared/api/query-client.ts` (toast-нотифікації)
- Індивідуально — `onError` у мутаціях для специфічної UI-логіки (виділення полів форми)
- `401` — ізольовано в axios interceptor (refresh + retry), не торкається TanStack Query

Конфігурація QueryClient, auth flow, BroadcastChannel logout — у [State Management](./state-management.md).

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
| Unit | Vitest | entities hooks, validation helpers, auth store, query keys |
| Integration | Vitest + RTL + MSW | форми, компоненти з API-викликами |
| E2E | Playwright | login flow, CRUD проєктів, workflow перекладача |

Розміщення тестів: `*.test.ts(x)` поряд з source файлами (co-location).

MSW handlers: `shared/test/msw/handlers/` — перевикористовуються у unit, integration тестах та dev-режимі.

**Coverage gate (CI):** `70%` — hard threshold, CI fails нижче. Ціль для entities/features: `80%`.

---

## Детальна документація

- [State Management](./state-management.md) — TanStack Query, Zustand, auth flow деталі
- [API Contracts](./api-contracts.md) — endpoints, формати відповідей, коди помилок
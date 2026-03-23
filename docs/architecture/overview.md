# Architecture Overview

Цей документ — відправна точка для розуміння архітектури. Читати перед будь-якими змінами до структури проєкту.

> **Для агентів:** архітектурні рішення зафіксовані. Змінювати стек, шари FSD або правила імпортів без явного дозволу людини — заборонено.

---

## Що це і як працює

TMS Frontend — SPA (Single Page Application) на React. Спілкується з REST API бекенду через HTTPS. Бекенд може знаходитись на окремому домені — фронтенд налаштований для роботи через CORS.

Додаток розгортається у Docker-контейнері з nginx, який роздає статичні файли після `vite build`. Це забезпечує незалежність від хостингової платформи.

```
┌──────────────────────────────────┐
│         Docker Container         │
│  nginx → /dist (vite build)      │
└──────────────────────────────────┘
               │ HTTPS
┌──────────────────────────────────┐
│         Backend API              │
│  /api/v1/  (окремий домен)       │
└──────────────────────────────────┘
```

---

## Чому Feature-Sliced Design

FSD обраний як основна архітектурна методологія з таких причин:

- **Масштабованість** — кожен домен (project, translation-key, user) живе у своєму slice і не змішується з іншими
- **Передбачуваність** — чіткі правила імпортів виключають circular dependencies
- **Придатність для AI-агентів** — агент може працювати з одним slice ізольовано, не ризикуючи зламати інші частини
- **Явний публічний API** — кожен slice експортує лише те, що дозволено через `index.ts`

---

## Шари FSD

Шари розташовані від найвищого рівня абстракції до найнижчого. Верхній шар може імпортувати з нижнього — але не навпаки.

```
app/        Провайдери, bootstrap, глобальні стилі. Точка входу додатку.
pages/      Сторінки маршрутів. Компонують widgets і features.
widgets/    Складені UI-блоки (AppShell, TranslationTable). Без бізнес-логіки запитів.
features/   Дії користувача з бізнес-логікою (create-project, edit-translation).
entities/   Доменні моделі: API-функції, TypeScript-типи, хуки TanStack Query.
shared/     Утиліти без бізнес-знань: axios instance, UI-primitives, i18n config.
```

**Правило імпортів:** `pages` → `widgets` → `features` → `entities` → `shared`. Зворотні імпорти заборонені. Enforced через `eslint-plugin-boundaries`.

---

## Ключові архітектурні рішення

### Стан додатку

Чотири рівні: server state (TanStack Query), client state (Zustand), form state (@mantine/form), URL state (TanStack Router search params). Серверні дані не дублюються у Zustand.

Детальні політики, query configuration, auth flow — у [State Management](./state-management.md).

### Аутентифікація

Access token зберігається in-memory (Zustand), refresh token — у httpOnly cookie. При 401 — axios interceptor виконує refresh і retry. Cross-tab logout — через BroadcastChannel.

Повна схема auth flow, promise queue, silent refresh — у [State Management](./state-management.md#auth-flow).

### Routing

TanStack Router з type-safe маршрутами. Auth guard реалізовано через `beforeLoad` на layout-маршруті `_authenticated` — не через окремі перевірки у компонентах.

### i18n

UI підтримує дві локалі: `en` та `uk`. Всі переклади зберігаються в одному файлі на локаль. RTL (`dir="rtl"`) встановлюється на input-ах для мов `ar`, `he`, `fa`, `ur` — визначається з API, не хардкодиться.

---

## Технологічний стек

| Категорія | Рішення | Чому |
|-----------|---------|------|
| Фреймворк | React 19 + Vite 6 + TypeScript 5 strict | Швидка збірка, типобезпека |
| UI | Mantine 8.x | Повне покриття UI-потреб, вбудована форма, модалки, нотифікації |
| Routing | TanStack Router v1 | Type-safe routes, вбудований auth guard через `beforeLoad` |
| Server state | TanStack Query v5 | Стандарт для REST: кеш, інвалідація, devtools |
| Client state | Zustand | Мінімальний бандл (~1KB), простий API |
| HTTP | axios | Interceptors для auth-retry, знайомий AI-агентам |
| Тести | Vitest + RTL + MSW / Playwright | Нативна інтеграція з Vite, реалістичний мокінг |
| Лінтинг | ESLint 9 + eslint-plugin-boundaries | Enforcement FSD import rules |

---

## Розгортання

```dockerfile
# Збірка
FROM node:22-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# Runtime
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

Конфігурація через змінні середовища у `.env`:

```
VITE_API_BASE_URL=https://api.yourdomain.com
```

> `VITE_*` змінні вбудовуються у білд на етапі збірки. Для зміни URL потрібен новий білд.

---

## Детальна документація

- [Frontend Architecture](./frontend-architecture.md) — структура папок, патерни slice, routing, конфіги
- [State Management](./state-management.md) — TanStack Query, Zustand, auth flow
- [API Contracts](./api-contracts.md) — endpoints, формати відповідей, коди помилок
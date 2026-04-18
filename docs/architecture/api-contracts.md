# API Contracts

Цей документ описує **як фронтенд взаємодіє з API**: розміщення TypeScript-типів, обгортку над **`fetch` / `ofetch`** у `shared/api`, auth і обробку помилок. Конвенції TanStack Query (key factories, staleTime, інвалідація) — у [State Management](./state-management.md).

> **Для агентів:** усі нові звернення до API — через **`shared/api`** (чисті функції запитів) і **TanStack Query** у `pages/*/queries` або `features/*` (або поруч із feature). **Не** викликати мережевий клієнт напряму з UI-компонентів. Архітектура шарів — **Page-First** (`app → pages → features → shared`); окремих шарів `entities/` / `widgets/` немає — див. [Overview](./overview.md) та [Folder Structure](../standards/folder-structure.md).

---

## Розміщення TypeScript-типів

- **Спільні транспортні типи** (`PaginatedResponse`, `ApiError`, типи тіл відповідей, які споживає кілька доменів) — у **`shared/api/`** (наприклад `shared/api/types.ts` або доменні файли на кшталт `project-types.ts`).
- **Доменні типи, прив’язані до одного екрана** — у мікромодулі сторінки: `pages/<area>/<page>/types/`.
- **Доменні типи спільної feature** — у `features/<name>/types.ts` (або папка `types/`), якщо feature експортує стабільний публічний API.

Приклади сутностей (канонічні поля узгоджені з [Data Models](./data-models.md) та [API Reference](./api-reference.md)):

```typescript
// shared/api/user-types.ts (приклад розміщення)
interface User {
  id: number;           // BigInt PK (не UUID!)
  email: string;
  first_name: string;
  last_name: string;
  is_active: boolean;
  is_staff: boolean;
  date_joined: string;  // ISO 8601
  last_login: string | null;
}
```

```typescript
// shared/api/project-types.ts (приклад)
interface Project {
  id: string;           // UUID
  slug: string;         // використовується в URL замість id
  name: string;
  description: string;
  created_at: string;   // ISO 8601
  updated_at: string;
  languages: ProjectLanguage[];
}

interface ProjectLanguage {
  id: string;
  language: string;          // код мови: "en", "uk"
  is_base_language: boolean;
  created_at: string;
}
```

```typescript
// shared/api/translation-key-types.ts (приклад)
interface TranslationKey {
  id: string;
  key: string;                              // dot-notation: "auth.login.title"
  description: string;
  translations: Record<string, string>;     // { "en": "Sign In", "uk": "" }
  created_at: string;
  updated_at: string;
}

interface TranslationValue {
  id: string;
  language: string;
  value: string;    // "" = не перекладено
  created_at: string;
  updated_at: string;
}
```

```typescript
// shared/api/types.ts
interface PaginatedResponse<T> {
  count: number;
  next: string | null;
  previous: string | null;
  results: T[];
}

interface ApiError {
  message: string;
  extra: {
    fields?: Record<string, string[]>;
    [key: string]: unknown;
  };
}
```

---

## HTTP-клієнт у `shared/api`

Єдина обгортка над **`ofetch`** або нативним **`fetch`** (базовий URL, заголовки, `credentials: 'include'` для httpOnly cookie на auth-ендпоінтах, `Authorization` з синхронного мосту токена). Рекомендована точка входу — модуль на кшталт `src/shared/api/client.ts` (точні імена файлів — у репозиторії).

### Доступ до access token поза React

Токен тримається **in-memory** у провайдері; HTTP-шар читає його через **`getAccessTokenSync()`** (див. [State Management — клієнтський стан / Auth](./state-management.md#клієнтський-стан-client-state)):

```typescript
import { getAccessTokenSync } from "@/shared/api/auth-session";

// У обгортці запиту перед відправкою:
const token = getAccessTokenSync();
const headers: HeadersInit = {
  ...(token ? { Authorization: `Bearer ${token}` } : {}),
};
```

### Поведінка при `401`

Одна спроба **refresh** через promise queue, потім **повтор** оригінального запиту. Якщо refresh невдалий — очищення сесії, за потреби **BroadcastChannel** для logout між вкладками та redirect на логін.

Повна схема — [State Management — Auth Flow](./state-management.md#auth-flow).

---

## TanStack Query — конвенції

Query key factories, staleTime, інвалідація, optimistic updates — у [State Management](./state-management.md#tanstack-query-server-state).

---

## Обробка помилок

### Формат помилки від API

```json
{
  "message": "Validation failed",
  "extra": {
    "fields": {
      "slug": ["This field must be unique."],
      "key": ["Conflict with existing key."]
    }
  }
}
```

### Утиліти для читання помилки

Працюють з **нормалізованою** помилкою, яку кидає/прокидає шар `shared/api` (об’єкт з полями на кшталт `status`, `data`), або з `unknown` після перевірки форми:

```typescript
// src/shared/api/errors.ts
function isApiErrorBody(data: unknown): data is ApiError {
  return (
    typeof data === "object" &&
    data !== null &&
    "message" in data &&
    typeof (data as ApiError).message === "string"
  );
}

export const getApiError = (error: unknown): string => {
  if (error && typeof error === "object" && "data" in error) {
    const data = (error as { data?: unknown }).data;
    if (isApiErrorBody(data)) return data.message;
  }
  if (isApiErrorBody(error)) return error.message;
  return "Network error";
};

export const getFieldErrors = (error: unknown): Record<string, string[]> => {
  if (error && typeof error === "object" && "data" in error) {
    const data = (error as { data?: unknown }).data;
    if (isApiErrorBody(data) && data.extra?.fields) return data.extra.fields;
  }
  if (isApiErrorBody(error) && error.extra?.fields) return error.extra.fields;
  return {};
};
```

### HTTP коди та реакція фронтенду

| Код | Ситуація | Реакція |
|-----|---------|---------|
| `400` | Помилка валідації | `getFieldErrors()` → поля форми; глобальний toast приглушити (`meta.suppressGlobalError` тощо — див. [State Management](./state-management.md)) |
| `401` | Не автентифікований | Обробка в `shared/api` → refresh → retry або logout |
| `403` | Немає прав | Повідомлення / redirect за політикою продукту |
| `404` | Ресурс не знайдений | На екрані очікувано — empty / 404 UI; інакше fallback за політикою |
| `409` | Конфлікт | Специфічний UI або локальний `onError` |
| `429` | Rate limit | Показати `Retry-After`, без зайвого автоматичного retry |
| `5xx` | Серверна помилка | Глобальний toast через `handleGlobalError`, якщо не приглушено локально |

### Нотифікації та глобальний шар

За замовчуванням помилки запитів/мутацій потрапляють у **`handleGlobalError`** на `QueryCache` / `MutationCache` ([State Management](./state-management.md)). Локально: **`onError`** у `useQuery` / `useMutation`, **`meta.suppressGlobalError: true`** для тихих або повністю оброблених кейсів, **`HandledApiError`** (або еквівалент), щоб глобальний шар не дублював toast.

Для кастомного повідомлення в конкретній мутації можна використати `@mantine/notifications` у `onError` — узгоджуючи з правилом **без дублювання** з [State Management](./state-management.md).

---

## Endpoint Map (зведена таблиця)

| Метод | URL | Опис |
|-------|-----|------|
| `POST` | `/api/v1/auth/token/` | Логін |
| `POST` | `/api/v1/auth/token/refresh/` | Refresh access token |
| `POST` | `/api/v1/auth/logout/` | Logout |
| `GET` | `/api/v1/users/` | Список користувачів |
| `GET` | `/api/v1/users/me/` | Поточний користувач |
| `PATCH` | `/api/v1/users/me/` | Оновити профіль |
| `POST` | `/api/v1/users/me/change-password/` | Змінити пароль |
| `GET` | `/api/v1/projects/` | Список проєктів |
| `POST` | `/api/v1/projects/` | Створити проєкт |
| `GET` | `/api/v1/projects/{slug}/` | Деталі проєкту |
| `PATCH` | `/api/v1/projects/{slug}/` | Оновити проєкт |
| `DELETE` | `/api/v1/projects/{slug}/` | Видалити проєкт |
| `GET` | `/api/v1/projects/{slug}/export/` | Експорт перекладів (auth) |
| `GET` | `/api/v1/projects/{slug}/languages/` | Мови проєкту |
| `POST` | `/api/v1/projects/{slug}/languages/` | Додати мову |
| `PATCH` | `/api/v1/projects/{slug}/languages/{lang}/` | Змінити базову мову |
| `DELETE` | `/api/v1/projects/{slug}/languages/{lang}/` | Видалити мову |
| `GET` | `/api/v1/projects/{slug}/keys/` | Список ключів (з фільтрами) |
| `POST` | `/api/v1/projects/{slug}/keys/` | Створити ключ |
| `GET` | `/api/v1/projects/{slug}/keys/{key}/` | Деталі ключа |
| `PATCH` | `/api/v1/projects/{slug}/keys/{key}/` | Оновити ключ |
| `DELETE` | `/api/v1/projects/{slug}/keys/{key}/` | Видалити ключ |
| `POST` | `/api/v1/projects/{slug}/keys/bulk-delete/` | Масово видалити ключі |
| `PATCH` | `/api/v1/projects/{slug}/keys/{key}/translations/` | Batch оновлення перекладів |
| `PUT` | `/api/v1/projects/{slug}/keys/{key}/translations/{lang}/` | Створити/замінити переклад |
| `DELETE` | `/api/v1/projects/{slug}/keys/{key}/translations/{lang}/` | Видалити переклад |
| `GET` | `/api/v1/languages/` | Всі мови системи (без auth) |
| `GET` | `/api/v1/public/{slug}/translations/` | Публічний експорт (без auth) |

> Детальний опис body, query params та відповідей — у [api-reference.md](./api-reference.md).

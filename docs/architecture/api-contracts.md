# API Contracts

Цей документ описує **як фронтенд взаємодіє з API**: TypeScript-типи, конфігурацію axios та обробку помилок. Конвенції TanStack Query (key factories, staleTime, інвалідація) — у [State Management](./state-management.md).

> **Для агентів:** нові запити до API реалізуються виключно через entities-шар FSD. Не викликати axios напряму з features або widgets.

---

## TypeScript-типи

Типи живуть у відповідних entity-слайсах у `src/entities/<entity>/model/types.ts`.

```typescript
// src/entities/user/model/types.ts
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
// src/entities/project/model/types.ts
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
// src/entities/translation-key/model/types.ts
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
// src/shared/api/types.ts
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

## axios Instance

Єдиний інстанс знаходиться у `src/shared/api/instance.ts`.

```typescript
const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  withCredentials: true,   // обов'язково — для передачі httpOnly cookie
});
```

### Request interceptor — додає Authorization header

```typescript
api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

### Response interceptor — 401 retry flow

При 401 — одна спроба refresh через promise queue, потім повтор оригінального запиту. Якщо refresh невдалий — `clearAuth()` + BroadcastChannel logout + redirect.

Повна схема auth flow, promise queue та BroadcastChannel — у [State Management](./state-management.md#auth-flow).

---

## TanStack Query — конвенції

Query key factories, staleTime стратегія, правила інвалідації кешу, оптимістичні оновлення — у [State Management](./state-management.md#tanstack-query-server-state).

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

### Утиліта для читання помилки

```typescript
// src/shared/api/errors.ts
function getApiError(error: unknown): string {
  if (isAxiosError(error) && error.response?.data) {
    return (error.response.data as ApiError).message ?? 'Unknown error';
  }
  return 'Network error';
}

function getFieldErrors(error: unknown): Record<string, string[]> {
  if (isAxiosError(error) && error.response?.data) {
    return (error.response.data as ApiError).extra?.fields ?? {};
  }
  return {};
}
```

### HTTP коди та реакція фронтенду

| Код | Ситуація | Реакція |
|-----|---------|---------|
| `400` | Помилка валідації | `getFieldErrors()` → показати біля поля |
| `401` | Не автентифікований | Interceptor → refresh → retry або logout |
| `403` | Немає прав | Показати повідомлення про відмову |
| `404` | Ресурс не знайдений | Redirect на сторінку 404 або повідомлення |
| `409` | Конфлікт (дублікат slug, конфлікт key) | Показати конкретну помилку |
| `429` | Rate limit | Показати `Retry-After`, не робити автоматичний retry |
| `5xx` | Серверна помилка | Загальне повідомлення, логувати в консоль |

### Нотифікації

Використовувати `notifications` з Mantine. **Не показувати toast при кожній помилці автоматично** — мутації показують помилку самостійно через `onError`.

```typescript
import { notifications } from '@mantine/notifications';

// В onError мутації
notifications.show({
  color: 'red',
  title: 'Помилка',
  message: getApiError(error),
});
```

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
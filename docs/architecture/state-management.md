# State Management

Canonical-документ для всього, що стосується стану додатку: серверного, клієнтського, форм, URL та auth flow.

> **Для агентів:** не змінювати стратегію управління станом, query policies або auth flow без явного дозволу людини. Всі рішення зафіксовані тут.

---

## Огляд: рівні стану

Стан додатку розділений на чотири незалежних рівні. Кожен рівень має своє джерело правди та інструмент.

| Рівень | Інструмент | Де живе | Scope |
|--------|-----------|---------|-------|
| Server state | TanStack Query v5 | `entities/<name>/api/`, `features/<name>/api/` | Дані з API: кешування, інвалідація, мутації |
| Client state | Zustand | `shared/auth/` | Auth токен + мінімальний UI-стан |
| Form state | @mantine/form | Локально у компоненті | Значення полів, валідація, dirty-tracking |
| URL state | TanStack Router search params | URL query string | Фільтрація, пагінація, view mode |

**Ключовий принцип:** дані з сервера **не дублюються** у Zustand. TanStack Query — єдине джерело правди для server data. Zustand зберігає лише те, що не є server state і не може бути в URL.

```
┌─────────────────────────────────────────────────┐
│                   Компонент                     │
│                                                 │
│  useQuery(...)        useAuthStore(...)          │
│  useMutation(...)     form.values               │
│                       route.search              │
└────────┬──────────────────┬──────────┬──────────┘
         │                  │          │
    TanStack Query      Zustand    TanStack Router
    (server data)    (auth token)  (URL params)
         │
     axios instance
         │
     Backend API
```

---

## TanStack Query (Server State)

### QueryClient конфігурація

```typescript
// src/shared/api/query-client.ts
import { QueryClient, QueryCache, MutationCache } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30 * 1000,
      gcTime: 5 * 60 * 1000,
      retry: 1,
      refetchOnWindowFocus: false,
    },
  },
  queryCache: new QueryCache({
    onError: (error) => handleGlobalError(error),
  }),
  mutationCache: new MutationCache({
    onError: (error) => handleGlobalError(error),
  }),
});
```

`handleGlobalError` — єдина точка показу toast-нотифікацій (`@mantine/notifications`):
- `429` — toast з повідомленням про ліміт
- `5xx` — toast "Server error"
- інші — toast з повідомленням з тіла відповіді або fallback

Індивідуальний `onError` у мутаціях — тільки для специфічної UI-логіки (виділити поле форми). Загальні toast-и не дублювати.

> `401` обробляється **не тут**, а в axios interceptor — TanStack Query бачить лише успішний повторний запит або помилку після retry. Деталі — у секції [Auth Flow](#auth-flow).

### Query Key Factory

Кожен entity має свій key factory у `src/entities/<entity>/model/query-keys.ts`. Патерн — обʼєкт з методами, що повертають `readonly` tuple.

```typescript
// src/entities/project/model/query-keys.ts
export const projectKeys = {
  all: ['projects'] as const,
  lists: () => [...projectKeys.all, 'list'] as const,
  detail: (slug: string) => [...projectKeys.all, 'detail', slug] as const,
  languages: (slug: string) => [...projectKeys.detail(slug), 'languages'] as const,
};
```

```typescript
// src/entities/translation-key/model/query-keys.ts
export const translationKeyKeys = {
  all: (projectSlug: string) => ['projects', projectSlug, 'keys'] as const,
  lists: (projectSlug: string, params?: TranslationKeyParams) =>
    [...translationKeyKeys.all(projectSlug), 'list', params] as const,
  detail: (projectSlug: string, key: string) =>
    [...translationKeyKeys.all(projectSlug), 'detail', key] as const,
};
```

```typescript
// src/entities/user/model/query-keys.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  me: () => [...userKeys.all, 'me'] as const,
  detail: (id: number) => [...userKeys.all, 'detail', id] as const,
};
```

```typescript
// src/entities/language/model/query-keys.ts
export const languageKeys = {
  all: ['languages'] as const,
};
```

### staleTime стратегія

| Тип даних | `staleTime` | Обґрунтування |
|-----------|-------------|---------------|
| Список проєктів | `30s` | Змінюється рідко, але має бути актуальним |
| Деталі проєкту + мови | `60s` | Менш волатильні |
| Список ключів (з пагінацією) | `30s` | Часто змінюється іншими користувачами |
| Поточний користувач (`/me/`) | `5min` | Рідко змінюється, оновлюється після edit profile |
| Список мов системи (`/languages/`) | `Infinity` | Довідник — не змінюється під час сесії |

`staleTime` задається на рівні окремих хуків `useQuery`, а не глобально (окрім default `30s` у QueryClient).

### Інвалідація кешу після мутацій

Інвалідація має бути **точковою** — через query key factory. Широка інвалідація (`queryKey: ['projects']`) скидає весь кеш піддерева і забороняється.

| Мутація | Інвалідація |
|---------|-------------|
| Створення/видалення проєкту | `projectKeys.lists()` |
| Оновлення проєкту | `projectKeys.detail(slug)` |
| Зміна мов проєкту | `projectKeys.detail(slug)` |
| Створення/видалення ключа | `translationKeyKeys.lists(projectSlug)` |
| Оновлення перекладу | `translationKeyKeys.lists(projectSlug)` |
| Bulk delete ключів | `translationKeyKeys.lists(projectSlug)` |
| Оновлення профілю | `userKeys.me()` |
| Зміна пароля | `userKeys.me()` + оновлення auth store |

```typescript
// Приклад: після створення проєкту
useMutation({
  mutationFn: createProject,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: projectKeys.lists() });
  },
});
```

### Оптимістичні оновлення

Застосовуються **тільки** для inline-редагування перекладів — де затримка критична для UX. Решта мутацій використовує стандартний flow (invalidation після success).

```typescript
const mutation = useMutation({
  mutationFn: updateTranslation,
  onMutate: async ({ key, lang, value }) => {
    await queryClient.cancelQueries({ queryKey: translationKeyKeys.lists(projectSlug) });
    const previous = queryClient.getQueryData(translationKeyKeys.lists(projectSlug));
    // optimistic update у кеші...
    return { previous };
  },
  onError: (_err, _vars, context) => {
    queryClient.setQueryData(translationKeyKeys.lists(projectSlug), context?.previous);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: translationKeyKeys.lists(projectSlug) });
  },
});
```

### Пагінація

Backend використовує `limit`/`offset`. Фронтенд оперує поняттям `page` (1-based) і конвертує:

```typescript
const offset = (page - 1) * limit;
```

```typescript
const { data, isPlaceholderData } = useQuery({
  queryKey: translationKeyKeys.lists(projectSlug, { search, lang, page, limit }),
  queryFn: () => fetchTranslationKeys(projectSlug, { search, lang, offset, limit }),
  placeholderData: keepPreviousData,
});
```

`placeholderData: keepPreviousData` — зберігає попередні дані під час завантаження нової сторінки, запобігаючи миготінню UI.

### Hooks conventions

| Тип | Розташування | Приклад |
|-----|-------------|---------|
| `useQuery` для entity | `entities/<name>/api/` | `use-projects.ts`, `use-project.ts` |
| `useMutation` для feature | `features/<name>/api/` | `use-create-project.ts` |
| Query key factory | `entities/<name>/model/` | `query-keys.ts` |
| API functions (axios calls) | `entities/<name>/api/` | `project-api.ts` |

Query хуки (`useQuery`) описують **зчитування даних** і належать entity-шару. Mutation хуки (`useMutation`) описують **дії користувача** і належать feature-шару.

---

## Zustand (Client State)

### Auth Store

Єдиний Zustand store у додатку. Зберігає мінімум того, що не є server state.

```typescript
// src/shared/auth/auth-store.ts
import { create } from 'zustand';

interface AuthState {
  accessToken: string | null;
  setAccessToken: (token: string) => void;
  clearAuth: () => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  accessToken: null,
  setAccessToken: (token) => set({ accessToken: token }),
  clearAuth: () => set({ accessToken: null }),
}));
```

Дані користувача (`User`) **не зберігаються** у Zustand — це server state, доступний через `useQuery` з ключем `userKeys.me()`.

### Принцип мінімалізму

Zustand зберігає лише:
- `accessToken` — JWT токен поточної сесії
- Можливі UI-преференції, якщо вони не можуть бути в URL (наприклад, стан sidebar)

Все інше має бути:
- **Server state** → TanStack Query
- **Filter/pagination state** → URL search params
- **Form state** → @mantine/form
- **Тема (dark/light)** → Mantine ColorScheme (localStorage через MantineProvider)

### Підписка через selectors

Для уникнення зайвих ре-рендерів — завжди використовувати селектор:

```typescript
// ✅ підписка лише на accessToken
const token = useAuthStore((state) => state.accessToken);

// ❌ підписка на весь store — ре-рендер при будь-якій зміні
const store = useAuthStore();
```

### Доступ поза React

Axios interceptor потребує токен поза React-деревом. Для цього використовується `getState()`:

```typescript
const token = useAuthStore.getState().accessToken;
```

Це синхронний виклик без підписки — підходить для interceptor-ів та утиліт.

---

## Form State (@mantine/form)

Form state — завжди локальний для компонента. Не використовувати Zustand або контекст для форм.

### Серверні помилки у формі

Після невдалої мутації — маппити `ApiError.extra.fields` на помилки полів форми:

```typescript
import { getFieldErrors } from '@/shared/api';

const mutation = useMutation({
  mutationFn: createProject,
  onError: (error) => {
    const fieldErrors = getFieldErrors(error);
    Object.entries(fieldErrors).forEach(([field, messages]) => {
      form.setFieldError(field, messages[0]);
    });
  },
});
```

### Валідація

Фронтенд-валідація дублює обмеження бекенду:
- slug: `^[a-z0-9]+(?:-[a-z0-9]+)*$`, max 100
- key: `^[a-z0-9]+(?:[._][a-z0-9]+)*$`, мінімум 2 сегменти
- password: мінімум 8 символів

Серверна валідація залишається авторитетною — фронтенд-валідація лише покращує UX.

---

## URL State

Параметри фільтрації та пагінації зберігаються в URL query string через TanStack Router search params. Це забезпечує deep linking, sharing URL та коректну роботу кнопки "назад".

### Search params для `/translations`

| Параметр | Тип | Default | Опис                                       |
|----------|-----|---------|--------------------------------------------|
| `search` | `string` | `''`    | Пошук за назвою ключа                      |
| `lang` | `string` | `''`    | Фільтр за мовою                            |
| `status` | `'translated' \| 'untranslated'` | `''`      | Статус перекладу (лише у зв'язці із мовою) |
| `sort` | `string` | `''`    | Сортування                                 |
| `page` | `number` | `1`     | Номер сторінки (1-based)                   |
| `limit` | `number` | `20`    | Елементів на сторінці                      |
| `view` | `'table' \| 'list'` | `'table'` | Режим відображення                         |

### Синхронізація з TanStack Query

URL params передаються в query key — при зміні URL автоматично створюється новий запит:

```typescript
function TranslationsPage() {
  const { search, lang, status, page, limit } = Route.useSearch();

  const { data } = useQuery({
    queryKey: translationKeyKeys.lists(projectSlug, { search, lang, status, page, limit }),
    queryFn: () => fetchTranslationKeys(projectSlug, {
      search, lang,
      untranslated: status === 'untranslated',
      offset: (page - 1) * limit,
      limit,
    }),
    placeholderData: keepPreviousData,
  });
}
```

---

## Auth Flow

### Зберігання токенів

| Токен | Де зберігається | TTL | Доступ з JS | Scope |
|-------|----------------|-----|-------------|-------|
| Access token (JWT) | Zustand (in-memory) | 60 хв | Так | Одна вкладка |
| Refresh token | httpOnly cookie | 7 днів | Ні | Всі вкладки (одне джерело) |

Access token **не зберігається** у localStorage і не в cookie — лише в памʼяті. Кожна вкладка має свій незалежний екземпляр.

### Повна схема

```
Login
  │
  ├── POST /auth/token/ { email, password }
  │     ├── 200 → { access } + Set-Cookie: refresh_token
  │     │         setAccessToken(access)
  │     │         redirect → /app
  │     └── 401 → показати помилку логіну
  │
Silent Refresh (при завантаженні додатку)
  │
  ├── POST /auth/token/refresh/ (cookie автоматично)
  │     ├── 200 → { access } + new cookie
  │     │         setAccessToken(access)
  │     └── 401 → redirect → /auth/login
  │
Authenticated Request
  │
  ├── request interceptor → додає Authorization: Bearer <token>
  │
  ├── 200 → дані повертаються у TanStack Query
  │
  └── 401 → response interceptor:
        ├── POST /auth/token/refresh/
        │     ├── 200 → setAccessToken(new) → retry original request
        │     └── fail → clearAuth() + BroadcastChannel + redirect /auth/login
        └── (promise queue: один refresh на всі паралельні 401)

Logout
  │
  ├── POST /auth/logout/ (blacklist refresh token)
  ├── clearAuth()
  ├── queryClient.clear()
  ├── BroadcastChannel → 'logout'
  └── redirect → /auth/login

Change Password
  │
  ├── POST /users/me/change-password/
  │     ├── 200 → { access } + new cookie
  │     │         setAccessToken(access)
  │     │         invalidate userKeys.me()
  │     └── 400 → показати помилки полів
```

### Promise Queue для паралельних 401

Коли кілька запитів одночасно отримують 401 — тільки перший ініціює refresh. Решта чекають на результат того самого promise:

```typescript
let refreshPromise: Promise<string> | null = null;

async function refreshAccessToken(): Promise<string> {
  if (refreshPromise) return refreshPromise;

  refreshPromise = axios
    .post('/api/v1/auth/token/refresh/', null, { withCredentials: true })
    .then(({ data }) => {
      useAuthStore.getState().setAccessToken(data.access);
      return data.access;
    })
    .finally(() => {
      refreshPromise = null;
    });

  return refreshPromise;
}
```

### BroadcastChannel — cross-tab logout

При logout або невдалому refresh — повідомлення надсилається всім вкладкам:

```typescript
const authChannel = new BroadcastChannel('tms-auth');

// Надсилання (при logout або failed refresh)
authChannel.postMessage({ type: 'logout' });

// Прослуховування (при mount додатку)
authChannel.addEventListener('message', (event) => {
  if (event.data.type === 'logout') {
    useAuthStore.getState().clearAuth();
    // redirect → /auth/login
  }
});
```

### Silent Refresh при mount

При першому завантаженні додатку (або refresh сторінки) — access token відсутній. Auth guard у TanStack Router `beforeLoad` ініціює silent refresh:

1. `beforeLoad` перевіряє `useAuthStore.getState().accessToken`
2. Якщо `null` — викликає `POST /auth/token/refresh/`
3. Якщо cookie валідний — отримує новий access token, продовжує
4. Якщо cookie невалідний — redirect на `/auth/login`

---

## Антипатерни

| Антипатерн | Чому заборонено | Правильно |
|-----------|-----------------|-----------|
| Дублювати server data у Zustand | Два джерела правди, розсинхронізація | Використовувати TanStack Query |
| React Context для глобального стану | Ре-рендер всього піддерева при зміні | Zustand (selector-based підписка) |
| Access token у localStorage | XSS-вразливість, доступний всім скриптам | In-memory (Zustand) |
| `invalidateQueries({ queryKey: ['projects'] })` | Скидає весь кеш піддерева | `invalidateQueries({ queryKey: projectKeys.lists() })` |
| Логіка refresh у TanStack Query callbacks | Змішує transport-рівень з data-рівнем | Ізольовано в axios interceptor |
| Зберігати filter state у Zustand | Втрачається при refresh, не можна поділитись URL | URL search params через TanStack Router |
| `useAuthStore()` без селектора | Зайві ре-рендери при зміні будь-якого поля | `useAuthStore((s) => s.accessToken)` |

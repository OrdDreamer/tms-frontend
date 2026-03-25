# State Management

Canonical-документ для всього, що стосується стану додатку: серверного, клієнтського, форм, URL та auth flow.

> **Для агентів:** не змінювати стратегію управління станом, query policies або auth flow без явного дозволу людини. Всі рішення зафіксовані тут.

---

## Огляд: рівні стану

Стан додатку розділений на чотири незалежних рівні. Кожен рівень має своє джерело правди та інструмент.

| Рівень       | Інструмент                                            | Де живе                                                                           | Scope                                                                |
| ------------ | ----------------------------------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Server state | TanStack Query v5                                     | `pages/*/queries`, `features/*/queries` (і `shared/api` для чистих fetch-функцій) | Дані з API: кешування, інвалідація, мутації                          |
| Client state | `useState` / `useReducer`, React Context (провайдери) | `app/providers`, `shared/auth` тощо                                               | Токен сесії, рідкісний глобальний UI-стан без дублювання server data |
| Form state   | @mantine/form                                         | Локально у компоненті                                                             | Значення полів, валідація, dirty-tracking                            |
| URL state    | TanStack Router search params                         | URL query string                                                                  | Фільтрація, пагінація, view mode                                     |

**Ключовий принцип:** дані з сервера **не дублюються** у глобальному клієнтському store. TanStack Query — єдине джерело правди для server data. Клієнтський шар тримає лише те, що не є server state і не може бути в URL. **Zustand як дефолтний вибір не рекомендується**; допустимий лише після окремого обґрунтування.

```
┌─────────────────────────────────────────────────┐
│                   Компонент                     │
│                                                 │
│  useQuery(...)        useAuth() / Context       │
│  useMutation(...)     form.values               │
│                       route.search              │
└────────┬──────────────────┬──────────┬──────────┘
         │                  │          │
    TanStack Query    Client state   TanStack Router
    (server data)     (провайдери)   (URL params)
         │
     shared/api (fetch / ofetch)
         │
     Backend API
```

---

## TanStack Query (Server State)

### QueryClient конфігурація

```typescript
// src/shared/api/query-client.ts
import { QueryClient, QueryCache, MutationCache } from "@tanstack/react-query";

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

`handleGlobalError` — **за замовчуванням** єдина точка toast-ів (`@mantine/notifications`) для помилок, які **не перехоплені локально** і **не позначені як тихі**.

#### Глобально (типові коди і кейси)

Показувати загальний toast / єдиний стиль повідомлення, якщо немає спеціальної локальної обробки:

| Код / ситуація                                           | Поведінка глобально                                                                        |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Мережа** (`failed to fetch`, timeout, offline)         | Один зрозумілий toast («немає мережі», «таймаут»)                                          |
| **429**                                                  | Toast про перевищення ліміту / rate limit                                                  |
| **502, 503, 504** (і загалом **5xx**)                    | Toast на кшталт «сервер недоступний» / server error                                        |
| **403** (якщо немає окремого екрана «доступ заборонено») | Короткий toast або redirect за політикою продукту                                          |
| **404** для «загальних» запитів (рідко)                  | Fallback toast; якщо це **очікувана** ситуація на конкретному екрані — див. локально нижче |
| **Інші 4xx / невідомі**                                  | Toast з текстом з тіла відповіді API або узгоджений fallback                               |

**`401`** у глобальний toast **зазвичай не виносимо**: спочатку refresh/retry у `shared/api`, потім при остаточній відмові — redirect на логін (можливо без «шумного» toast). Деталі — у секції [Auth Flow](#auth-flow).

#### Локально (пріоритет над глобальним)

Якщо для запиту/мутації потрібна **своя** логіка — обробляй у `onError` на `useQuery` / `useMutation` (або в обгортці):

- **400** з **валідаційними полями** (`extra.fields` / подібне) — мапінг на `form.setFieldError`, **без** дублюючого глобального toast (познач контекст як оброблений або `meta: { suppressGlobalError: true }`, якщо впровадите такий прапорець).
- **409, 422** — конфлікт / бізнес-правило: часто потрібен **специфічний** текст або UI (модалка), не загальний toast.
- **404** на екрані, де «не знайдено» — **очікуваний** стан (показ порожнього стану / 404-сторінки), а не глобальний toast.
- **Тихі** фонові запити (prefetch тощо) — без toast; передати **`meta.suppressGlobalError: true`** (або еквівалент), щоб `handleGlobalError` їх пропустив.

#### Правило без дублювання

Якщо помилку вже оброблено локально — глобальний шар **не повинен** показувати другий toast. Домовленість проєкту: або прапорець у `meta`, або узгоджений тип помилки `HandledApiError`, який `handleGlobalError` ігнорує.

### Query Key Factory

Усі key factory для TanStack Query живуть у **`shared/query-keys/`** — по одному файлу на домен даних (наприклад `project-keys.ts`, `user-keys.ts`). `pages/*` та `features/*` **тільки імпортують** фабрики з `shared`, а не тримають власні копії ключів.

**Навіщо так:** одна форма `queryKey` для читання та для `invalidateQueries` / optimistic update; **немає крос-імпортів** між фічами й сторінками заради доступу до ключів.

Патерн — обʼєкт з методами, що повертають `readonly` tuple:

```typescript
// shared/query-keys/project-keys.ts
export const projectKeys = {
  all: ["projects"] as const,
  lists: () => [...projectKeys.all, "list"] as const,
  detail: (slug: string) => [...projectKeys.all, "detail", slug] as const,
  languages: (slug: string) =>
    [...projectKeys.detail(slug), "languages"] as const,
};
```

```typescript
// shared/query-keys/translation-key-keys.ts
export const translationKeyKeys = {
  all: (projectSlug: string) => ["projects", projectSlug, "keys"] as const,
  lists: (projectSlug: string, params?: TranslationKeyParams) =>
    [...translationKeyKeys.all(projectSlug), "list", params] as const,
  detail: (projectSlug: string, key: string) =>
    [...translationKeyKeys.all(projectSlug), "detail", key] as const,
};
```

```typescript
// shared/query-keys/user-keys.ts
export const userKeys = {
  all: ["users"] as const,
  lists: () => [...userKeys.all, "list"] as const,
  me: () => [...userKeys.all, "me"] as const,
  detail: (id: number) => [...userKeys.all, "detail", id] as const,
};
```

```typescript
// shared/query-keys/language-keys.ts
export const languageKeys = {
  all: ["languages"] as const,
};
```

### staleTime стратегія

| Тип даних                          | `staleTime` | Обґрунтування                                    |
| ---------------------------------- | ----------- | ------------------------------------------------ |
| Список проєктів                    | `30s`       | Змінюється рідко, але має бути актуальним        |
| Деталі проєкту + мови              | `60s`       | Менш волатильні                                  |
| Список ключів (з пагінацією)       | `30s`       | Часто змінюється іншими користувачами            |
| Поточний користувач (`/me/`)       | `5min`      | Рідко змінюється, оновлюється після edit profile |
| Список мов системи (`/languages/`) | `Infinity`  | Довідник — не змінюється під час сесії           |

`staleTime` задається на рівні окремих хуків `useQuery`, а не глобально (окрім default `30s` у QueryClient).

### Інвалідація кешу після мутацій

Інвалідація має бути **точковою** — через query key factory. Широка інвалідація (`queryKey: ['projects']`) скидає весь кеш піддерева і забороняється.

| Мутація                     | Інвалідація                                                                             |
| --------------------------- | --------------------------------------------------------------------------------------- |
| Створення/видалення проєкту | `projectKeys.lists()`                                                                   |
| Оновлення проєкту           | `projectKeys.detail(slug)`                                                              |
| Зміна мов проєкту           | `projectKeys.detail(slug)`                                                              |
| Створення/видалення ключа   | `translationKeyKeys.lists(projectSlug)`                                                 |
| Оновлення перекладу         | `translationKeyKeys.lists(projectSlug)`                                                 |
| Bulk delete ключів          | `translationKeyKeys.lists(projectSlug)`                                                 |
| Оновлення профілю           | `userKeys.me()`                                                                         |
| Зміна пароля                | `userKeys.me()` + оновлення access token (сесія через провайдер / `setAccessTokenSync`) |

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
    await queryClient.cancelQueries({
      queryKey: translationKeyKeys.lists(projectSlug),
    });
    const previous = queryClient.getQueryData(
      translationKeyKeys.lists(projectSlug),
    );
    // optimistic update у кеші...
    return { previous };
  },
  onError: (_err, _vars, context) => {
    queryClient.setQueryData(
      translationKeyKeys.lists(projectSlug),
      context?.previous,
    );
  },
  onSettled: () => {
    queryClient.invalidateQueries({
      queryKey: translationKeyKeys.lists(projectSlug),
    });
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
  queryKey: translationKeyKeys.lists(projectSlug, {
    search,
    lang,
    page,
    limit,
  }),
  queryFn: () =>
    fetchTranslationKeys(projectSlug, { search, lang, offset, limit }),
  placeholderData: keepPreviousData,
});
```

`placeholderData: keepPreviousData` — зберігає попередні дані під час завантаження нової сторінки, запобігаючи миготінню UI.

### Hooks conventions

| Тип                                              | Розташування                                      | Приклад                                           |
| ------------------------------------------------ | ------------------------------------------------- | ------------------------------------------------- |
| `useQuery` / `useMutation`, прив’язані до екрану | `pages/<area>/<page>/queries/`                    | `use-project-list.ts`                             |
| Спільні між сторінками query/mutation-хуки       | `features/<name>/queries/` (або поруч із feature) | `use-create-project.ts`                           |
| Query key factory                                | `shared/query-keys/` (один файл на домен)         | `project-keys.ts`, `user-keys.ts`                 |
| Чисті функції HTTP (без React)                   | `shared/api/`                                     | `project-api.ts`, обгортка над `fetch` / `ofetch` |

Читання та зміна даних на бекенді йдуть через TanStack Query: **`queryFn` / `mutationFn`** викликають функції з `shared/api`, а не роблять мережевий виклик прямо в компоненті.

---

## Клієнтський стан (Client state)

### Підхід

- **Локально** — `useState` / `useReducer` у компоненті або в невеликому піддереві.
- **Спільно для багатьох гілок UI** — один або кілька **вузьких** React Context (провайдери): мемоізований `value`, розділення контекстів, щоб уникати зайвих ре-рендерів.
- **Zustand** — не рекомендується як стандарт; якщо колись з’явиться в проєкті, це має бути явне архітектурне рішення з обґрунтуванням.

### Auth: токен і доступ з HTTP-шару

Токен сесії тримається **in-memory** у провайдері (або в невеликому модулі, синхронізованому з ним). Дані користувача (`User`) — це **server state** (`useQuery`, наприклад `userKeys.me()`), а не поле глобального клієнтського store.

Для `401` / refresh HTTP-обгортці потрібен **синхронний** доступ до поточного access token поза React. Типовий патерн: невеликий модуль-мір (`getAccessToken` / `setAccessToken`), який оновлюється з `AuthProvider` при логіні / logout / refresh:

```typescript
// shared/api/auth-session.ts — приклад контракту (реалізація в проєкті)
let accessToken: string | null = null;

export const setAccessTokenSync = (token: string | null) => {
  accessToken = token;
};

export const getAccessTokenSync = () => accessToken;
```

Провайдер при зміні токена викликає `setAccessTokenSync`, щоб `shared/api` міг додавати `Authorization` і керувати чергою refresh без підписки на React.

### Принцип мінімалізму

Глобальний клієнтський шар зберігає лише необхідний мінімум, наприклад:

- access token (сесія)
- винятково ті UI-прапорці, які не в URL і не є server state (рідко)

Усе інше:

- **Server state** → TanStack Query
- **Фільтри / пагінація** → URL search params
- **Форми** → @mantine/form
- **Тема (dark/light)** → Mantine ColorScheme (через `MantineProvider`)

---

## Form State (@mantine/form)

Form state — завжди локальний для компонента. Не виносити стан форми у глобальний контекст чи окремий store (крім локального контексту самої форми, якщо це навмисний UX).

### Серверні помилки у формі

Після невдалої мутації — маппити `ApiError.extra.fields` на помилки полів форми:

```typescript
import { getFieldErrors } from "@/shared/api";

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

| Параметр | Тип                              | Default   | Опис                                       |
| -------- | -------------------------------- | --------- | ------------------------------------------ |
| `search` | `string`                         | `''`      | Пошук за назвою ключа                      |
| `lang`   | `string`                         | `''`      | Фільтр за мовою                            |
| `status` | `'translated' \| 'untranslated'` | `''`      | Статус перекладу (лише у зв'язці із мовою) |
| `sort`   | `string`                         | `''`      | Сортування                                 |
| `page`   | `number`                         | `1`       | Номер сторінки (1-based)                   |
| `limit`  | `number`                         | `20`      | Елементів на сторінці                      |
| `view`   | `'table' \| 'list'`              | `'table'` | Режим відображення                         |

### Синхронізація з TanStack Query

URL params передаються в query key — при зміні URL автоматично створюється новий запит:

```typescript
function TranslationsPage() {
  const { search, lang, status, page, limit } = Route.useSearch();

  const { data } = useQuery({
    queryKey: translationKeyKeys.lists(projectSlug, {
      search,
      lang,
      status,
      page,
      limit,
    }),
    queryFn: () =>
      fetchTranslationKeys(projectSlug, {
        search,
        lang,
        untranslated: status === "untranslated",
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

| Токен              | Де зберігається                                   | TTL    | Доступ з JS | Scope                      |
| ------------------ | ------------------------------------------------- | ------ | ----------- | -------------------------- |
| Access token (JWT) | In-memory (React Provider + синх-модуль для HTTP) | 60 хв  | Так         | Одна вкладка               |
| Refresh token      | httpOnly cookie                                   | 7 днів | Ні          | Всі вкладки (одне джерело) |

Access token **не зберігається** у localStorage і не в cookie — лише в памʼяті. Кожна вкладка має свій незалежний екземпляр.

### Прагматичний баланс: простота vs безпека

Ця схема навмисно **проста** і **достатньо безпечна** для типового SPA + REST:

- **Refresh у httpOnly** — JS не може прочитати довгоживучий секрет → суттєво краще, ніж `localStorage` / `sessionStorage`.
- **Access in-memory** — після hard reload потрібен один **silent refresh** (невелика ціна), зате немає «залиплого» access у сховищі і вужче вікно ризику при XSS порівняно з токеном у LS.
- **Альтернатива «обидва токени в cookies»** зручніша після F5 (рідше видно затримку), але вимагає ретельнішої **CSRF / SameSite / CORS** дисципліни на бекенді.
- **localStorage для токенів** не рекомендується як усталений вибір: при XSS зловмисник може **ексфільтрувати** довгу сесію. Допустиме лише як свідомий компроміс (наприклад, закриті внутрішні інструменти) з короткими TTL і оцінкою ризиків.

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

  // apiPostRefresh() — реалізація в shared/api (POST /auth/token/refresh/, credentials: 'include')
  refreshPromise = apiPostRefresh()
    .then((data) => {
      setAccessTokenSync(data.access);
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
const authChannel = new BroadcastChannel("tms-auth");

// Надсилання (при logout або failed refresh)
authChannel.postMessage({ type: "logout" });

// Прослуховування (при mount додатку)
authChannel.addEventListener("message", (event) => {
  if (event.data.type === "logout") {
    setAccessTokenSync(null);
    // redirect → /auth/login
  }
});
```

### Silent Refresh при mount

При першому завантаженні додатку (або refresh сторінки) — access token відсутній. Auth guard у TanStack Router `beforeLoad` ініціює silent refresh:

1. `beforeLoad` перевіряє наявність access token (наприклад `getAccessTokenSync()` або стан провайдера)
2. Якщо `null` — викликає `POST /auth/token/refresh/`
3. Якщо cookie валідний — отримує новий access token, продовжує
4. Якщо cookie невалідний — redirect на `/auth/login`

---

## Антипатерни

| Антипатерн                                       | Чому заборонено                                   | Правильно                                                         |
| ------------------------------------------------ | ------------------------------------------------- | ----------------------------------------------------------------- |
| Дублювати server data у глобальному client store | Два джерела правди, розсинхронізація              | TanStack Query                                                    |
| Один великий Context на весь додаток             | Зайві ре-рендери, складно підтримувати            | Вузькі провайдери, мемоізований `value`, стан ближче до споживача |
| Access token у localStorage                      | XSS-вразливість, доступний всім скриптам          | In-memory + httpOnly refresh cookie                               |
| `invalidateQueries({ queryKey: ['projects'] })`  | Скидає весь кеш піддерева                         | `invalidateQueries({ queryKey: projectKeys.lists() })`            |
| Логіка refresh у TanStack Query callbacks        | Змішує transport-рівень з data-рівнем             | Ізольовано в `shared/api`                                         |
| Зберігати filter state у глобальному store       | Втрачається при refresh, не шариться URL          | URL search params через TanStack Router                           |
| Рекламація Zustand «на все»                      | Зайва залежність при наявності Query + URL + форм | Спочатку Query, URL, локальний стан і провайдери                  |

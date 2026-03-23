# API Reference

Base URL: `http://localhost:8000` (dev) | `https://yourdomain.com` (prod)

Всі endpoints (крім позначених) потребують заголовок:
```
Authorization: Bearer <access_token>
```

---

## Authentication

> Refresh token передається через httpOnly cookie. Всі запити до `/api/v1/auth/` потребують `credentials: 'include'` / `withCredentials: true`. Детально — [auth.md](auth.md).

### `POST /api/v1/auth/token/`
Отримати токени (логін).

- **Rate limit**: 10 req/хв
- **Auth**: Не потрібна

**Body:**
```json
{ "email": "user@example.com", "password": "password123" }
```

**200:**
```json
{ "access": "..." }
```

> Refresh token встановлюється як httpOnly cookie (`Set-Cookie: refresh_token=...; Path=/api/v1/auth/; HttpOnly`).

---

### `POST /api/v1/auth/token/refresh/`
Оновити access token.

- **Auth**: Не потрібна
- **Body**: Не потрібне — refresh token зчитується з httpOnly cookie

**200:**
```json
{ "access": "..." }
```

> Новий refresh token автоматично встановлюється у cookie (rotation).

**401** — cookie відсутній або токен невалідний/прострочений.

---

### `POST /api/v1/auth/logout/`
Завершити сесію (занести refresh token у blacklist, очистити cookie).

- **Body**: Не потрібне — refresh token зчитується з cookie

**204:** No Content

---

## Users

### `GET /api/v1/users/`
Список користувачів.

**Query params:**
| Параметр | Тип | Опис |
|---------|-----|------|
| `search` | string | Фільтр за email, first_name, last_name |
| `limit` | int | Кількість (за замовчуванням 20, макс 100) |
| `offset` | int | Зміщення |

**200:** [`PaginatedResponse<User>`](data-models.md#paginated-response)

---

### `GET /api/v1/users/{id}/`
Профіль конкретного користувача.

**200:** [`User`](data-models.md#user)

---

### `GET /api/v1/users/me/`
Профіль поточного користувача.

**200:** [`User`](data-models.md#user)

---

### `PATCH /api/v1/users/me/`
Оновити профіль поточного користувача.

**Body (усі поля опціональні):**
```json
{ "first_name": "Іван", "last_name": "Петренко" }
```

**200:** [`User`](data-models.md#user)

---

### `POST /api/v1/users/me/change-password/`
Змінити пароль. **Інвалідує всі токени.**

**Body:**
```json
{
  "current_password": "...",
  "new_password": "...",
  "new_password_confirm": "..."
}
```

**200:**
```json
{ "access": "..." }
```

> Новий refresh token встановлюється у cookie. Оновіть access token у store.

---

## Projects

### `GET /api/v1/projects/`
Список проєктів.

**Query params:**
| Параметр | Тип | Опис |
|---------|-----|------|
| `limit` | int | За замовчуванням 20, макс 100 |
| `offset` | int | Зміщення |

**200:** [`PaginatedResponse<Project>`](data-models.md#project)

---

### `POST /api/v1/projects/`
Створити проєкт.

**Body:**
```json
{
  "slug": "my-app",
  "name": "My Application",
  "description": "Optional"
}
```

**Обмеження slug**: `a-z`, `0-9`, `-`; унікальний; max 100 символів.

**201:** [`Project`](data-models.md#project)

---

### `GET /api/v1/projects/{slug}/`
Деталі проєкту (включаючи список мов).

**200:** [`Project`](data-models.md#project)

---

### `PATCH /api/v1/projects/{slug}/`
Оновити проєкт.

**Body (часткове оновлення):**
```json
{ "name": "New Name", "description": "Updated" }
```

**200:** [`Project`](data-models.md#project)

---

### `DELETE /api/v1/projects/{slug}/`
Видалити проєкт (каскадно видаляє всі мови, ключі, переклади).

**204:** No Content

---

### `GET /api/v1/projects/{slug}/export/`
Експортувати переклади проєкту (authenticated версія).

**Query params:**
| Параметр | Тип | Опис |
|---------|-----|------|
| `lang` | string | Конкретна мова (опціонально) |
| `export_format` | `flat` \| `nested` | За замовчуванням `flat` |

**200:**
```json
{
  "en": { "auth.login.title": "Sign In", "form.submit": "Submit" },
  "uk": { "auth.login.title": "Вхід", "form.submit": "Надіслати" }
}
```

---

## Project Languages

### `GET /api/v1/projects/{slug}/languages/`
Список мов проєкту.

**200:** `ProjectLanguage[]`

---

### `POST /api/v1/projects/{slug}/languages/`
Додати мову до проєкту.

**Body:**
```json
{ "language": "de", "is_base_language": false }
```

> Якщо це перша мова — стає базовою автоматично незалежно від `is_base_language`.

**201:** [`ProjectLanguage`](data-models.md#projectlanguage)

---

### `PATCH /api/v1/projects/{slug}/languages/{lang_code}/`
Зробити мову базовою.

**Body:**
```json
{ "is_base_language": true }
```

> Попередня базова мова автоматично втрачає статус базової.

**200:** [`ProjectLanguage`](data-models.md#projectlanguage)

---

### `DELETE /api/v1/projects/{slug}/languages/{lang_code}/`
Видалити мову (і всі її переклади).

**Обмеження:**
- Не можна видалити базову мову (треба спочатку призначити іншу)
- Не можна видалити останню мову проєкту

**204:** No Content

---

## Translation Keys

### `GET /api/v1/projects/{slug}/keys/`
Список ключів перекладів.

**Query params:**
| Параметр | Тип | Опис |
|---------|-----|------|
| `search` | string | Фільтр за назвою ключа |
| `lang` | string | Фільтр — лише ключі з цією мовою |
| `untranslated` | bool | Лише ключі БЕЗ перекладу для `lang` |
| `include_translations` | bool | Включити переклади у відповідь (за замовчуванням `true`) |
| `limit` | int | За замовчуванням 20, макс 100 |
| `offset` | int | Зміщення |

**200:** [`PaginatedResponse<TranslationKey>`](data-models.md#translationkey)

---

### `POST /api/v1/projects/{slug}/keys/`
Створити ключ (з опціональними перекладами).

**Body:**
```json
{
  "key": "auth.login.title",
  "description": "Login page title",
  "translations": {
    "en": "Sign In",
    "uk": "Вхід"
  }
}
```

**Обмеження key:**
- Pattern: `^[a-z0-9]+(?:[._][a-z0-9]+)*$`
- Мінімум 2 сегменти: `auth.title` ✓, `auth` ✗
- Тільки малі літери, цифри, `.`, `_`
- Не може конфліктувати з існуючими: якщо є `menu`, не можна додати `menu.file`

**201:** [`TranslationKey`](data-models.md#translationkey)

---

### `GET /api/v1/projects/{slug}/keys/{key_name}/`
Деталі ключа з усіма перекладами.

> `key_name` — це сам ключ, напр. `auth.login.title` (не UUID)

**200:** [`TranslationKey`](data-models.md#translationkey)

---

### `PATCH /api/v1/projects/{slug}/keys/{key_name}/`
Оновити ключ.

**Body:**
```json
{ "description": "Updated description" }
```

**200:** [`TranslationKey`](data-models.md#translationkey)

---

### `DELETE /api/v1/projects/{slug}/keys/{key_name}/`
Видалити ключ та всі його переклади.

**204:** No Content

---

### `POST /api/v1/projects/{slug}/keys/bulk-delete/`
Масово видалити ключі.

**Body:**
```json
{ "keys": ["auth.login.title", "auth.login.subtitle", "form.submit"] }
```

**200:** `{}`

---

## Translations

### `GET /api/v1/projects/{slug}/keys/{key}/translations/`
Список перекладів для ключа.

**200:** `TranslationValue[]`

---

### `PATCH /api/v1/projects/{slug}/keys/{key}/translations/`
Batch оновити кілька перекладів одночасно.

**Body:**
```json
{
  "translations": {
    "en": "Sign In",
    "uk": "Вхід",
    "de": ""
  }
}
```
> Якщо `value = ""` — переклад видаляється (якщо існував).

**200:** `TranslationValue[]`

---

### `PUT /api/v1/projects/{slug}/keys/{key}/translations/{lang_code}/`
Створити або замінити переклад для мови.

**Body:**
```json
{ "value": "Sign In" }
```

**200 або 201:** [`TranslationValue`](data-models.md#translationvalue)

---

### `DELETE /api/v1/projects/{slug}/keys/{key}/translations/{lang_code}/`
Видалити переклад для конкретної мови.

**204:** No Content

---

## Public API (без аутентифікації)

### `GET /api/v1/public/{slug}/translations/`
Публічний експорт перекладів проєкту.

- **Auth**: Не потрібна
- **Rate limit**: 150 req/хв
- **Cache**: ETag підтримка, `Cache-Control: public, max-age=60`

**Query params:**
| Параметр | Тип | Опис |
|---------|-----|------|
| `lang` | string | Тільки одна мова |
| `export_format` | `flat` \| `nested` | За замовчуванням `flat` |

Детально — [public-api.md](public-api.md).

---

## Languages

### `GET /api/v1/languages/`
Список усіх підтримуваних мов у системі.

- **Auth**: Не потрібна
- **Rate limit**: Відсутній

**200:**
```json
[
  { "code": "uk", "name": "Українська" },
  { "code": "en", "name": "English" },
  { "code": "es", "name": "Español" },
  ...
]
```

> Використовуйте цей endpoint замість хардкоду списку мов на фронтенді — список може змінюватись на бекенді.


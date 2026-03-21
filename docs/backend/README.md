# TMS Frontend Documentation

Translation Management System — документація для розробки фронтенду.

## Зміст

| Файл | Опис |
|------|------|
| [api-reference.md](api-reference.md) | Повний довідник API endpoints |
| [data-models.md](data-models.md) | TypeScript-типи та бізнес-правила |

## Що це за система

TMS — централізована система управління перекладами:

- **Projects** — проєкти (web-app, mobile-app тощо)
- **Languages** — мови проєкту (одна базова)
- **Keys** — ключі перекладів у dot-notation (`auth.login.title`)
- **Translations** — значення ключа для кожної мови

## Base URL та префікси

```
Development:  http://localhost:8000
Production:   https://yourdomain.com
```

```
/api/v1/        — authenticated endpoints
/api/v1/public/ — публічний API (без авторизації)
/api/v1/auth/   — авторизація
```

## Авторизація

- **Access token** — TTL 60 хв, передається у заголовку `Authorization: Bearer <token>`
- **Refresh token** — TTL 7 днів, зберігається у **httpOnly cookie** (недоступний через JS)
- **Token rotation** — кожен `/token/refresh/` видає нову пару; старий refresh token відразу стає недійсним

Всі запити до `/api/v1/auth/` потребують `credentials: 'include'` (fetch) або `withCredentials: true` (axios) — без цього браузер не надсилатиме cookie.

Після зміни пароля (`POST /api/v1/users/me/change-password/`) — всі токени інвалідуються. Відповідь повертає нову пару у тілі/cookie.

## Rate limits

| Endpoint | Ліміт |
|---------|-------|
| `POST /api/v1/auth/token/` | 10 req/хв |
| Authenticated endpoints | 300 req/хв |
| Anonymous endpoints | 60 req/хв |
| Public translations API | 150 req/хв |

При перевищенні — **HTTP 429** із заголовком `Retry-After`.

## Пагінація

LimitOffset: параметри `limit` (за замовчуванням 20, макс 100) та `offset`.

```json
{
  "count": 321,
  "next": "http://localhost:8000/api/v1/.../keys/?limit=20&offset=20",
  "previous": null,
  "results": [...]
}
```

`count` — **загальна** кількість. `next`/`previous` — абсолютні URL або `null`.

## Формат помилок

```json
{ "message": "Опис помилки", "extra": {} }
```

Для помилок валідації (400) `extra.fields` містить деталі по полях:

```json
{ "message": "Validation error", "extra": { "fields": { "slug": ["Project with this slug already exists."] } } }
```

### HTTP статус-коди

| Статус | Коли |
|--------|------|
| `200` | Успіх |
| `201` | Ресурс створено |
| `204` | Успішне видалення |
| `304` | Не змінилось (ETag) |
| `400` | Помилка валідації |
| `401` | Не аутентифіковано / токен протух |
| `403` | Немає прав |
| `404` | Не знайдено |
| `429` | Rate limit |
| `500` | Серверна помилка |

### Бізнес-помилки

| Ситуація | Статус | Деталі |
|---------|--------|--------|
| Slug вже зайнятий | 400 | `extra.fields.slug` |
| Видалення базової / останньої мови | 400 | `message` |
| Невалідний формат або конфлікт ключа | 400 | `extra.fields.key` або `message` |

### Правила валідації на фронті

| Поле | Правило |
|------|---------|
| Slug проєкту | `^[a-z0-9]+(?:-[a-z0-9]+)*$`, max 100 символів |
| Ключ перекладу | `^[a-z0-9]+(?:[._][a-z0-9]+)*$`, мінімум 2 сегменти через `.` |
| Пароль | Мінімум 8 символів |

## RTL мови

Для мов нижче встановлюйте `dir="rtl"` у layout:

| Код | Мова |
|-----|------|
| `ar` | العربية |
| `he` | עברית |
| `fa` | فارسی |
| `ur` | اردو |

> Повний список мов — `GET /api/v1/languages/`. Не хардкодьте на фронтенді.

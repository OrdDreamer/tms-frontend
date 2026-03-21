# Моделі даних

TypeScript-інтерфейси для контрактів API.

---

## User

```typescript
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

---

## Language

```typescript
interface Language {
  code: string;  // ISO 639-1 або BCP 47 ("en", "zh-CN")
  name: string;  // локальна назва ("English", "简体中文")
}
```

---

## Project

```typescript
interface Project {
  id: string;           // UUID
  slug: string;         // унікальний, використовується в URL замість UUID
  name: string;
  description: string;
  created_at: string;   // ISO 8601
  updated_at: string;
  languages: ProjectLanguage[];
}
```

---

## ProjectLanguage

```typescript
interface ProjectLanguage {
  id: string;
  language: string;          // код мови
  is_base_language: boolean;
  created_at: string;
}
```

**Бізнес-правила:**
- Перша додана мова автоматично стає базовою
- `PATCH { "is_base_language": true }` — призначає базовою; попередня базова втрачає статус автоматично
- Не можна видалити базову мову та останню мову проєкту

---

## TranslationKey

```typescript
interface TranslationKey {
  id: string;
  key: string;                              // dot-notation: "auth.login.title"
  description: string;
  translations: Record<string, string>;     // { "en": "Sign In", "uk": "" }
  created_at: string;
  updated_at: string;
}
```

**Формат ключа:** `^[a-z0-9]+(?:[._][a-z0-9]+)*$`, мінімум 2 сегменти через крапку.

**Конфлікт вкладеності:** якщо існує ключ `menu`, не можна додати `menu.file` — і навпаки (400).

---

## TranslationValue

```typescript
interface TranslationValue {
  id: string;
  language: string;
  value: string;    // "" = не перекладено
  created_at: string;
  updated_at: string;
}
```

Batch update: `value = ""` → запис видаляється.

---

## PaginatedResponse

```typescript
interface PaginatedResponse<T> {
  count: number;          // загальна кількість (не розмір сторінки)
  next: string | null;
  previous: string | null;
  results: T[];
}
```

---

## ApiError

```typescript
interface ApiError {
  message: string;
  extra: {
    fields?: Record<string, string[]>;  // деталі по полях при 400
    [key: string]: unknown;
  };
}
```

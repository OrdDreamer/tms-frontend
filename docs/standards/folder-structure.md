# Folder Structure

Операційний стандарт для розміщення файлів у `src/`. Документ визначає **де саме створювати код** та як уникати хаотичної структури.

> Canonical-джерело архітектурних рішень: `docs/architecture/frontend-architecture.md`. Якщо виникає конфлікт, пріоритет має architecture-документація.

---

## Scope

- Цей документ описує правила розміщення файлів і папок.
- Документ не змінює FSD-шари, імпортні межі та state-стратегію.
- Пов'язані джерела:
  - `docs/architecture/overview.md`
  - `docs/architecture/frontend-architecture.md`
  - `docs/architecture/state-management.md`
  - `docs/standards/coding-guidelines.md`

---

## 1) Базове дерево `src/`

```text
src/
├── app/
├── pages/
├── widgets/
├── features/
├── entities/
└── shared/
```

### Rule: Не створювати нові top-level папки поруч із шарами FSD
**Why:** зберігається єдина точка навігації і передбачувана структура.

**Example**
```text
✅ src/features/create-project/
❌ src/modules/create-project/
```

---

## 2) Шари без slice-ів: `app/` і `shared/`

### Rule: `app/` містить глобальний bootstrap (без бізнес-сутностей)
**Why:** старт додатку має бути централізованим і ізольованим від доменної логіки.

**Рекомендована структура**
```text
app/
├── providers/
├── styles/
└── index.tsx
```

### Rule: `shared/` містить перевикористовуваний код без бізнес-знань
**Why:** спільний фундамент не повинен залежати від доменів.

**Рекомендована структура**
```text
shared/
├── api/
├── auth/
├── config/
├── i18n/
├── ui/
├── lib/
└── test/
```

---

## 3) Шари зі slice-ами: `pages/`, `widgets/`, `features/`, `entities/`

### Rule: Кожна папка на цих шарах — окремий slice із публічним API
**Why:** чіткі модульні межі і контрольований доступ ззовні.

**Example**
```text
entities/project/
├── api/
├── model/
└── index.ts
```

### Rule: Прямі імпорти з внутрішніх файлів slice заборонені
**Why:** внутрішню структуру можна змінювати без каскадних правок у проєкті.

**Example**
```ts
// ✅
import { useProjects } from '@/entities/project';

// ❌
import { useProjects } from '@/entities/project/api/use-projects';
```

---

## 4) Внутрішня структура slice

Базовий шаблон (додавати тільки потрібні сегменти):

```text
<slice-name>/
├── ui/
├── api/
├── model/
├── lib/
├── config/
└── index.ts
```

### Rule: Не створювати порожні сегменти "на майбутнє"
**Why:** порожні папки ускладнюють огляд і вводять в оману.

### Rule: Назви сегментів за призначенням (`api`, `model`, `ui`)
**Why:** однакова навігація для всіх slice і агентів.

---

## 5) Placement Rules by Artifact

### Rule: Query hooks і API-виклики
**Why:** transport-логіка має бути централізована на нижчих шарах.

**Placement**
- `entities/<entity>/api/*` — `useQuery`, fetch-функції, query adapters
- `features/<feature>/api/*` — `useMutation` для user action
- `shared/api/*` — axios instance, базові API-утиліти

### Rule: Типи, query keys, модельні утиліти
**Why:** доменна модель не повинна змішуватись з UI.

**Placement**
- `entities/<entity>/model/types.ts`
- `entities/<entity>/model/query-keys.ts`
- `features/<feature>/model/*` (за потреби)

### Rule: UI-артефакти
**Why:** відокремлення презентаційного шару від бізнес-логіки.

**Placement**
- `ui/` всередині поточного slice для локального UI
- `shared/ui/` для перевикористовуваних примітивів
- `widgets/` для великих складених блоків сторінки

### Rule: Утиліти
**Why:** уникнення "utils-звалищ" і прихованих залежностей.

**Placement**
- `shared/lib/*` — загальні утиліти без бізнес-специфіки
- `<slice>/lib/*` — утиліти, що обслуговують конкретний slice

---

## 6) Тести і test infrastructure

### Rule: Unit/Integration тести розміщувати поряд із кодом (co-location)
**Why:** тест живе поруч із поведінкою, яку перевіряє.

**Example**
```text
entities/project/api/use-projects.ts
entities/project/api/use-projects.test.ts
```

### Rule: Спільні MSW/тест-адаптери тримати в `shared/test/*`
**Why:** одне місце для перевикористання тестової інфраструктури.

---

## 7) Імпортні межі і залежності

### Rule: Дотримуватися напрямку шарів
**Why:** запобігання циклам залежностей і architectural drift.

**Напрямок**
`pages` -> `widgets` -> `features` -> `entities` -> `shared`

### Rule: Cross-slice зв'язки мінімізувати
**Why:** менший coupling і простіша еволюція модулів.

**Дозволено**
- через публічний API (`index.ts`)
- лише коли зв'язок семантично виправданий

---

## 8) Anti-Patterns Checklist

- створення папок поза FSD-шарами
- прямі імпорти з внутрішніх файлів slice
- змішування UI, transport і model в одному файлі
- `shared/utils` як "усе підряд"
- дублювання однакових компонентів у різних slice замість `shared/ui`
- порожні сегменти `api/model/ui` "про запас"
- створення feature-логіки у `pages/` замість `features/`

---

## 9) Structure Review Checklist

Перед merge перевір:

- нові файли лежать у правильному FSD-шарі
- у кожного нового slice є `index.ts` як public API
- відсутні прямі імпорти у внутрішні файли slice
- `useQuery`/API-функції в `entities/*/api`, `useMutation` у `features/*/api`
- тести розташовані поряд із кодом або у `shared/test/*` для інфраструктури

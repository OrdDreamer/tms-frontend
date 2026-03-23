# Agent Rules

Агенто-незалежні правила для AI-розробки в проєкті. Документ застосовується до будь-якого інструмента/агента (Cursor, Claude, ChatGPT, Copilot та ін.).

---

## Scope

- Визначає ролі агентів, межі змін і обов'язковий контекст перед роботою.
- Фіксує протоколи оновлення прогресу (`docs/tasks/*`) і рішень (`decisions.md`).
- Не дублює архітектуру та стандарти: посилається на canonical docs.

## Canonical Sources (обов'язково читати перед стартом)

- `docs/architecture/overview.md`
- `docs/architecture/frontend-architecture.md`
- `docs/architecture/state-management.md`
- `docs/architecture/api-contracts.md`
- `docs/standards/coding-guidelines.md`
- `docs/standards/folder-structure.md`
- `docs/standards/naming-conventions.md`
- `docs/standards/ui-patterns.md`

Для конкретної задачі додатково:
- відповідний `docs/features/<feature>/spec.md`
- відповідний `docs/features/<feature>/tasks.md`
- відповідний `docs/features/<feature>/decisions.md` (якщо існує)

---

## Ролі агентів

### Architect

Відповідає за:
- формування/оновлення feature spec
- декомпозицію spec -> tasks
- фіксацію архітектурно значущих рішень у decisions

Не виконує:
- реалізацію всього обсягу коду без задачного поділу

### Implementer

Відповідає за:
- реалізацію **однієї** атомарної задачі з `tasks.md`
- оновлення статусу задач у `docs/tasks/*`
- локальне оновлення decisions (коли з'явилось нове рішення)

Не виконує:
- зміни architecture/standards без дозволу людини

### Reviewer

Відповідає за:
- перевірку diff на відповідність architecture/standards/spec
- виявлення ризиків, регресій, пропущених тестів
- рекомендації для виправлення перед завершенням задачі

---

## Що можна змінювати без окремого дозволу

- Код, тести, локальні feature-файли, що прямо стосуються задачі.
- `docs/features/<feature>/tasks.md` (прогрес по задачах фічі).
- `docs/features/<feature>/decisions.md` (фіксація рішення і why).
- `docs/tasks/backlog.md`, `docs/tasks/in-progress.md`, `docs/tasks/done.md` (глобальний статус).

## Що заборонено змінювати без дозволу людини

- `docs/architecture/*`
- `docs/standards/*`
- стратегія state management, auth flow, API integration rules
- базові правила FSD/імпортів та інші інваріанти архітектури

---

## Протокол перед початком задачі

1. Прочитати canonical docs (мінімум architecture + standards).
2. Прочитати конкретний `spec.md` і вибрану задачу в `tasks.md`.
3. Уточнити `Input / Output / Constraints` задачі.
4. Підтвердити, що задача атомарна і має критерії завершення.

---

## Протокол оновлення прогресу (global Kanban)

1. Перед стартом:
   - перенести задачу/фічу в `docs/tasks/in-progress.md`
   - зазначити відповідального агента
2. Після завершення:
   - перенести пункт у `docs/tasks/done.md`
   - за потреби залишити короткий контекст (що саме закрито)
3. Якщо не почато:
   - тримати у `docs/tasks/backlog.md`

---

## Протокол фіксації рішень (decision log)

Кожне нетривіальне рішення у фічі фіксується в `decisions.md` у форматі:
- Decision
- Status
- Why
- Impact

Мета: наступний агент має розуміти, чому зроблено саме так, без відновлення контексту з чату.

---

## Вимоги до якості виконання

- Дотримання `docs/standards/*` обов'язкове.
- Задачі мають бути атомарні, з чітким входом/виходом.
- Не дублювати правила з architecture/standards, а посилатися на них.
- У змінах має бути прозорий зв'язок: `spec -> task -> code/tests -> review -> done`.

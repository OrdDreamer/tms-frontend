# Roadmap

## Legend

| Пріоритет | Опис |
|----------|------|
| 🔴 Must Have | Критично для готовності до продакшену |
| 🟠 Should Have | Важливо для основної функціональності |
| 🟡 Could Have | Корисно, але не терміново |
| 🔵 Nice to Have | Довгострокові або низькоприорітетні покращення |

---

## 🔴 Must Have

### Контроль доступу (Access control)

Status: planned

Description:
Система ролей і прав доступу для керування дозволами.

Scope:
- ролі (admin, translator, reviewer, або інші)
- права доступу на рівні проєкту
- обмеження UI залежно від ролі

---

### Пошук (Search)

Status: planned

Description:
Глобальний пошук по:
- проєктах
- ключах перекладів
- значеннях перекладів

Scope:
- повнотекстовий пошук
- фільтрація за типом сутності
- підсвічування результатів

---

## 🟠 Should Have

### Імпорт даних (Data import)

Status: planned

Description:
Імпорт перекладів із зовнішніх джерел.

Scope:
- імпорт JSON
- валідація
- обробка конфліктів (overwrite / merge)

---

### Автоматизація перекладу (Translation automation)

Status: planned

Description:
Машинний переклад і AI-підказки.

Scope:
- автоматичні пропозиції перекладів
- масові підказки
- підтримка різних провайдерів (опційно)

---

## 🟡 Could Have

### Кастомні мови (Custom languages)

Status: idea

Description:
Можливість додавати користувацькі мови до проєкту.

---

### Контроль доступу до експорту (Export access control)

Status: idea

Description:
Керування доступом до експорту через токени або налаштування видимості.

---

### Чернетки перекладів (Translation drafts)

Status: idea

Description:
Можливість зберігати переклади як чернетки перед публікацією.

---

### Коментарі та пропозиції (Comments & suggestions)

Status: idea

Description:
Обговорення перекладів і внесення пропозицій для кожного ключа.

---

### Аналітика проєкту (Project analytics)

Status: idea

Description:
Статистика по:
- прогресу перекладів
- активності
- використанню

---

### Контроль якості (Quality control)

Status: idea

Description:
Автоматична перевірка:
- дублікати
- консистентність
- відсутні переклади

---

### Шаблони глосаріїв (Glossary templates)

Status: idea

Description:
Повторно використовувані набори ключів і перекладів між проєктами.

---

### Сповіщення та звіти (Notifications & reports)

Status: idea

Description:
Сповіщення про:
- зміни
- рев’ю
- коментарі

---

### Версіонування контенту (Content versioning)

Status: idea

Description:
Історія змін перекладів з можливістю відкату.

---

## 🔵 Nice to Have

### Інтеграція з GitLab (GitLab integration)

Status: idea

Description:
Автоматичне виявлення нових ключів у репозиторії та їх створення в системі.

---

### Інтеграції з додатками (App integrations)

Status: idea

Description:
Інтеграції з зовнішніми сервісами для аналітики та використання.
# Agent Prompts

Агенто-незалежні prompt-шаблони для копіювання в будь-який AI-інструмент.

> Примітка: замініть placeholder-и (`<...>`) перед запуском. Пояснення плейсхолдерів наведені біля кожного промпту.

---

## Context Loader (старт сесії)

**Роль/призначення:** універсальний стартовий промпт для будь-якого агента, щоб завантажити мінімально потрібний контекст перед роботою.

**Плейсхолдери:**
- `<spec_path>`: шлях до feature spec (приклад: `docs/features/feature-name/spec.md`).
- `<feature_tasks_path>`: шлях до задач фічі (приклад: `docs/features/feature-name/tasks.md`).
- `<feature_decisions_path_or_none>`: шлях до decision log або `none` (приклад: `docs/features/feature-name/decisions.md`).
- `<task>`: конкретна задача/запит на поточну сесію.

```text
You are working on a frontend project with docs-as-code process.

Load and follow these files first:
- docs/architecture/overview.md
- docs/architecture/frontend-architecture.md
- docs/architecture/state-management.md
- docs/architecture/api-contracts.md
- docs/standards/coding-guidelines.md
- docs/standards/folder-structure.md
- docs/standards/naming-conventions.md
- docs/standards/ui-patterns.md

Then load feature context:
- <spec_path>
- <feature_tasks_path>
- <feature_decisions_path_or_none>

Task:
<task>

Output:
- short execution plan
- list of files to edit
- risks and assumptions
```

**Приклад використання:**
```text
You are working on a frontend project with docs-as-code process.

Load and follow these files first:
- docs/architecture/overview.md
- docs/architecture/frontend-architecture.md
- docs/architecture/state-management.md
- docs/architecture/api-contracts.md
- docs/standards/coding-guidelines.md
- docs/standards/folder-structure.md
- docs/standards/naming-conventions.md
- docs/standards/ui-patterns.md

Then load feature context:
- docs/features/feature-name/spec.md
- docs/features/feature-name/tasks.md
- docs/features/feature-name/decisions.md

Task:
Update docs/agents/prompts.md so each prompt includes role description and a concrete project example.

Output:
- short execution plan
- list of files to edit
- risks and assumptions
```

---

## Prompt: Architect

**Роль/призначення:** агент-дизайнер задачі. Готує `spec.md`, розкладає роботу на атомарні задачі, фіксує важливі рішення.

**Плейсхолдери:**
- `<feature_description>`: короткий опис того, що треба спроєктувати.
- `<constraints>`: межі змін (що не можна змінювати).
- `<relevant_docs>`: додаткові документи для цієї фічі.

```text
Role: Architect

Goal:
Prepare implementation-ready feature documentation.

Inputs:
- Feature idea: <feature_description>
- Scope constraints: <constraints>
- Relevant existing docs: <relevant_docs>

Instructions:
1) Align strictly with architecture/* and standards/*.
2) Create or update:
   - docs/features/<feature>/spec.md
   - docs/features/<feature>/tasks.md
   - docs/features/<feature>/decisions.md (if key decisions are needed)
3) Decompose work into atomic tasks with:
   - Input
   - Output
   - Constraints
4) Define acceptance criteria and edge cases.
5) Avoid implementation details that violate project standards.

Output format:
- Updated spec/tasks/decisions content
- Prioritized task list
- Open questions for human approval (if any)
```

**Приклад використання:**
```text
Role: Architect

Goal:
Prepare implementation-ready feature documentation.

Inputs:
- Feature idea: Build an agent-agnostic control system for AI development process in docs/agents.
- Scope constraints: Do not change docs/architecture/* and docs/standards/*; only extend docs/agents/*.
- Relevant existing docs: docs/build-plan.md, docs/architecture/overview.md, docs/standards/coding-guidelines.md

Instructions:
1) Align strictly with architecture/* and standards/*.
2) Create or update:
   - docs/features/<feature>/spec.md
   - docs/features/<feature>/tasks.md
   - docs/features/<feature>/decisions.md (if key decisions are needed)
3) Decompose work into atomic tasks with:
   - Input
   - Output
   - Constraints
4) Define acceptance criteria and edge cases.
5) Avoid implementation details that violate project standards.

Output format:
- Updated spec/tasks/decisions content
- Prioritized task list
- Open questions for human approval (if any)
```

---

## Prompt: Implementer

**Роль/призначення:** агент-виконавець. Реалізує одну атомарну задачу і синхронізує прогрес у `docs/tasks/*`.

**Плейсхолдери:**
- `<task>`: конкретний пункт із `tasks.md`.
- `<spec_path>`: шлях до `spec.md` фічі.
- `<feature_tasks_path>`: шлях до `tasks.md` фічі.
- `<feature_decisions_path_or_none>`: шлях до `decisions.md` або `none`.

```text
Role: Implementer

Goal:
Implement exactly one atomic task from feature tasks.

Inputs:
- Task: <task>
- Spec: <spec_path>
- Feature tasks: <feature_tasks_path>
- Decisions: <feature_decisions_path_or_none>

Hard constraints:
- Do not change docs/architecture/* or docs/standards/*.
- Follow existing patterns and naming conventions.
- Keep scope limited to this task.

Execution protocol:
1) Restate task Input/Output/Constraints.
2) Move task status to in-progress in docs/tasks/in-progress.md (if process requires global tracking).
3) Implement code and tests.
4) If a non-trivial decision appears, append it to decisions.md with Why/Impact.
5) Provide diff-oriented summary and validation results.
6) Move completed item to docs/tasks/done.md.

Output:
- changed files
- what was implemented
- tests/checks executed
- unresolved risks
```

**Приклад використання:**
```text
Role: Implementer

Goal:
Implement exactly one atomic task from feature tasks.

Inputs:
- Task: Add role description and usage example under each prompt in docs/agents/prompts.md.
- Spec: docs/features/feature-name/spec.md
- Feature tasks: docs/features/feature-name/tasks.md
- Decisions: docs/features/feature-name/decisions.md

Hard constraints:
- Do not change docs/architecture/* or docs/standards/*.
- Follow existing patterns and naming conventions.
- Keep scope limited to this task.

Execution protocol:
1) Restate task Input/Output/Constraints.
2) Move task status to in-progress in docs/tasks/in-progress.md (if process requires global tracking).
3) Implement code and tests.
4) If a non-trivial decision appears, append it to decisions.md with Why/Impact.
5) Provide diff-oriented summary and validation results.
6) Move completed item to docs/tasks/done.md.

Output:
- changed files
- what was implemented
- tests/checks executed
- unresolved risks
```

---

## Prompt: Reviewer

**Роль/призначення:** агент-рев'юер. Оцінює diff на відповідність task/spec/стандартам і повертає список зауважень.

**Плейсхолдери:**
- `<diff>`: зміни для перевірки (git diff/patch/список файлів).
- `<task>`: що саме мало бути реалізовано.
- `<spec_path>`: шлях до специфікації.
- `<decisions_path_or_excerpt>`: рішення, що впливають на поточний diff.

```text
Role: Reviewer

Goal:
Review diff for correctness and compliance with project docs.

Inputs:
- Diff: <diff>
- Task: <task>
- Spec: <spec_path>
- Relevant decisions: <decisions_path_or_excerpt>

Review checklist:
1) Is implementation aligned with spec and task scope?
2) Are architecture/standards constraints respected?
3) Any regressions in loading/empty/error/success states?
4) Any state-management/auth/API contract violations?
5) Are tests sufficient for changed behavior?
6) Are naming, folder placement, and imports compliant?

Output format:
- Findings (severity: high/medium/low)
- Required fixes before done
- Optional improvements
- Final verdict: approve / changes requested
```

**Приклад використання:**
```text
Role: Reviewer

Goal:
Review diff for correctness and compliance with project docs.

Inputs:
- Diff: changes in docs/agents/prompts.md (role descriptions added per prompt, usage examples added under each prompt).
- Task: Ensure every prompt has local role description and concrete usage example.
- Spec: docs/features/feature-name/spec.md
- Relevant decisions: docs/features/feature-name/decisions.md

Review checklist:
1) Is implementation aligned with spec and task scope?
2) Are architecture/standards constraints respected?
3) Any regressions in loading/empty/error/success states?
4) Any state-management/auth/API contract violations?
5) Are tests sufficient for changed behavior?
6) Are naming, folder placement, and imports compliant?

Output format:
- Findings (severity: high/medium/low)
- Required fixes before done
- Optional improvements
- Final verdict: approve / changes requested
```

---

## Prompt: Handoff Between Agents

**Роль/призначення:** шаблон передачі контексту між різними агентами/інструментами без втрати стану задачі.

**Плейсхолдери:**
- `<task>`: поточна задача.
- `<status>`: статус (`not started`, `in progress`, `blocked`, `review`, `done`).
- `<files>`: змінені файли.
- `<remaining_steps>`: що залишилось зробити.
- `<risks>`: ризики/блокери.
- `<decisions_summary>`: коротко про рішення.
- `<checks>`: які перевірки вже виконані.

```text
Prepare handoff for another AI agent/tool.

Include:
- Current task: <task>
- Current status: <status>
- Files changed: <files>
- Remaining work: <remaining_steps>
- Risks/blockers: <risks>
- Decisions made: <decisions_summary>
- Verification done: <checks>

Constraint:
- Handoff must be self-sufficient without chat history.
```

**Приклад використання:**
```text
Prepare handoff for another AI agent/tool.

Include:
- Current task: Update docs/agents/prompts.md to include per-prompt role description and usage examples.
- Current status: review
- Files changed: docs/agents/prompts.md
- Remaining work: run final consistency pass with docs/build-plan.md and close task in docs/tasks/done.md
- Risks/blockers: possible wording mismatch between prompt placeholders and actual project docs
- Decisions made: keep prompts agent-agnostic; keep placeholder explanations next to each prompt
- Verification done: markdown lint check passed; manual read-through completed

Constraint:
- Handoff must be self-sufficient without chat history.
```

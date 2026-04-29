---
name: git-branch
description: "Draft ADHOC-style git branch names following the pattern `<version>-<h|t>-<slug>-<nickname>`. Use when Codex needs to propose a branch name for a ticket or task, where version is the Odoo series (18.0, 19.0, ...), `h` indicates a ticket (helpdesk) and `t` indicates a task, slug is a short kebab-case summary of the change, and nickname is the user's initials (e.g. `joa`, `jjs`, `lav`, `jok`)."
---

# Git Branch

## Overview

Draft branch names that match the ADHOC convention.

Pattern:

```text
<version>-<h|t>-<slug>-<nickname>
```

Where:

- `version`: Odoo series, e.g. `18.0`, `19.0`.
- `h | t`: `h` for ticket (helpdesk), `t` for task.
- `slug`: short kebab-case summary of the change. Lowercase, ASCII, words separated by `-`.
- `nickname`: user's initials, e.g. `joa`, `jjs`, `lav`, `jok`.

Do not create or checkout the branch unless the user explicitly asks for it.

## Workflow

1. Gather inputs.
- Ask the user for the missing pieces if any of `version`, type (`h` or `t`), slug source, or `nickname` is unclear.
- If the user mentions a ticket, use `h`. If they mention a task, use `t`.
- If the user provides a ticket/task description, derive the slug from it.

2. Inspect repository context for hints when useful.
- Use `git branch --show-current` and `git log --oneline -n 20` to infer the active version or recent naming style.
- Use `git branch -a` to confirm the prefix conventions in use.

3. Build the slug.
- Lowercase, ASCII only.
- Replace spaces and punctuation with `-`.
- Collapse repeated `-` and trim leading/trailing `-`.
- Keep it short and specific. Prefer 3 to 6 words.
- Do not include the module name unless it adds clarity and is not already implied by the version or scope.

4. Assemble the branch name.
- Format: `<version>-<h|t>-<slug>-<nickname>`
- Example for a ticket on 18.0 by `joa`: `18.0-h-fix-invoice-rounding-joa`
- Example for a task on 19.0 by `lav`: `19.0-t-add-product-import-lav`

## Output

When the user asks for a suggested branch name, return:

```text
<version>-<h|t>-<slug>-<nickname>
```

If any field is ambiguous, state the assumption briefly and still provide the best draft.

When the user explicitly asks to create the branch:

- Confirm the final name first.
- Use `git checkout -b <branch>` from the appropriate base.
- Stop and ask if the base branch is unclear.

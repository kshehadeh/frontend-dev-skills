# Next Cache Components Migration Skills

This repository contains a Factory skill for diagnosing and fixing Next.js `cacheComponents: true` migration issues.

## Repository Layout

```text
skills/
  next-cache-components-migration/
    SKILL.md
    core-concepts.md
    decision-framework.md
    common-patterns.md
    debugging.md
    migration-checklist.md
```

## How to Use These Skills

1. Open `skills/next-cache-components-migration/SKILL.md` first.
2. Use it as the entry point to route to the right reference file for your issue.
3. Follow the linked docs by problem type:
   - `core-concepts.md`: cache model and handling strategies
   - `decision-framework.md`: how to choose the right fix
   - `common-patterns.md`: known failure patterns and fixes
   - `debugging.md`: stack trace and isolation workflow
   - `migration-checklist.md`: end-to-end migration steps

## Using This Skill with Droid

Factory discovers skills from `.factory/skills/<skill-name>/SKILL.md` (or `~/.factory/skills/` for personal scope).

To use this repo's skill in a project:

1. Create `.factory/skills/next-cache-components-migration/` in your target repo.
2. Copy the files from `skills/next-cache-components-migration/` into that directory.
3. Restart Droid so it rescans skills.
4. Ask Droid to help with Next.js cache components migration issues; it will load the skill when relevant.

Note: this skill sets `user-invocable: false`, so it is intended for automatic model invocation rather than manual `/skill-name` invocation.

## Invoking This Skill via Prompt

If you are prompting an LLM/agent directly, include explicit instructions to load this skill before solving the task.

Example prompt:

```text
Use the next-cache-components-migration skill.
Read SKILL.md first, then follow the linked section that matches this issue.
Then propose and apply the fix.

Issue: "Uncached data outside Suspense" on app/products/[slug]/page.tsx
```

This works best when you provide:

- the concrete error (for example: "Uncached data outside Suspense"), and
- the file(s)/route currently failing.

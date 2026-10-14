---
name: coding-style
description: Conventions for writing code that is easy to read, change and delete. Use when writing, refactoring or reviewing code in any language, with extra guides for TypeScript and React.
---

# Coding Style Guide

Conventions for writing code that is easy to read, change and delete. Written for humans and AI agents.

## The project comes first

This guide fills gaps. It never overrides the project you are working in.

- When the project has a convention, follow it: its formatter and linter config, its own style guide or `AGENTS.md`, and what its code consistently does.
- A convention is what the project does on purpose and across the codebase. One nearby file doing something is not a convention. Legacy code that breaks these rules is not a pattern to copy.
- Rules marked *(opinion)* are a preference, not a principle. Drop them freely when the project chooses otherwise.

## Which guide to read

| When you are... | Read |
| --- | --- |
| Writing or reviewing any code, in any language | [general_principles.md](references/general_principles.md) |
| Writing or reviewing TypeScript | [general_principles.md](references/general_principles.md), then [typescript.md](references/typescript.md) |
| Writing or reviewing React | [general_principles.md](references/general_principles.md), [typescript.md](references/typescript.md), then [react.md](references/react.md) |

Read the files that apply before you start, not after.

## The rules in one screen

1. One file, one job. One function, one job.
2. Depend downward only: entry points, then presentation, then orchestration, then pure logic.
3. Names carry the meaning. If a name needs a comment, rename it.
4. No comments by default. When justified, one line, saying why and never what.
5. Write the least code that is obviously correct. Delete before you add.
6. Prefer data (tables, registries, maps) over branching (`if` and `switch` chains).
7. Keep pure logic apart from side effects. Push side effects to the edges.
8. Test behavior through the public surface. Mock only at the system boundary.
9. Do not build for needs you do not have yet (YAGNI).
10. Leave every file you touch a little better than you found it.

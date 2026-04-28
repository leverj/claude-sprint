# Install — Gemini CLI

Gemini CLI loads `GEMINI.md` from the project root and supports `@`-imports.

## Install

```bash
git clone https://github.com/leverj/claude-sprint ~/sprint-workflow
```

In each project, create `GEMINI.md` containing:

```
@~/sprint-workflow/SKILL.md
```

## Invocation

```bash
gemini "run sprint: pick 2"
gemini "run sprint: status"
```

Text after `run sprint:` is `<USER REQUEST>`. `<SKILL DIR>` is the clone path.

## Update

```bash
cd ~/sprint-workflow && git pull
```

# Install — OpenCode

OpenCode auto-discovers `AGENTS.md` in the project root.

## Setup

```bash
git clone https://github.com/leverj/claude-sprint ~/sprint-workflow
cd /path/to/your/project
ln -s ~/sprint-workflow/AGENTS.md AGENTS.md
```

(If symlinks aren't available, copy: `cp ~/sprint-workflow/AGENTS.md AGENTS.md` and re-copy on update.)

## Invocation

```bash
opencode "run sprint: pick 2"
opencode "run sprint: plan add OAuth login"
```

The text after `run sprint:` is the `<USER REQUEST>`. `<SKILL DIR>` is the directory containing the symlinked `AGENTS.md`.

## Update

```bash
cd ~/sprint-workflow && git pull
```

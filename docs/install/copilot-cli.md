# Install — GitHub Copilot CLI

Copilot CLI treats `AGENTS.md` in the repo root as primary instructions (and additionally honors `.github/copilot-instructions.md`).

## Setup

```bash
git clone https://github.com/leverj/claude-sprint ~/sprint-workflow
cd /path/to/your/project
ln -s ~/sprint-workflow/AGENTS.md AGENTS.md
```

(If symlinks aren't available, copy: `cp ~/sprint-workflow/AGENTS.md AGENTS.md` and re-copy on update.)

## Invocation

```bash
copilot "run sprint: pick 2"
copilot "run sprint: status"
```

The text after `run sprint:` is the `<USER REQUEST>`. `<SKILL DIR>` is the directory containing the symlinked `AGENTS.md`.

## Update

```bash
cd ~/sprint-workflow && git pull
```

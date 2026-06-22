# Setup instructions for Claude

You are reading this because the user pointed you at this repository. Your job is to help them **install the skills they want** from it. Follow this protocol exactly — do not improvise an install.

## When the user asks to set up / install / add these skills (or just "set this up")

1. **Enter plan mode immediately.** Do **not** copy, write, register, or modify anything until the user approves a plan. Setup is a plan-first operation.
2. **Build the catalog.** Find every `SKILL.md` under `skills/` (they may be nested in category folders). For each, show its folder name and the one-line `description:` from its frontmatter. Group by category folder if present.
3. **Offer two selection modes and ask the user which they want:**
   - **Prescriptive** — they pick specific skills by name or number (accept lists and ranges), **or**
   - **Select all** — install every skill.
   If `skills/references/` exists, treat it as shared dependency files (e.g. `dt-brand` needs them) — include it automatically whenever a skill that depends on it is selected, or with "all".
4. **Present the plan via ExitPlanMode.** Spell out: the exact skills to install, that each `skills/<name>/` copies to `~/.claude/skills/<name>/` (and `skills/references/` → `~/.claude/skills/references/` if included), and **explicitly flag any skill that already exists and would be overwritten**.
5. **Only after approval, apply it:** copy the selected folders into `~/.claude/skills/` (honor `$CLAUDE_CONFIG_DIR/skills` if that env var is set). Create the target dir if missing. Never touch skills the user didn't choose.
6. **Report** what was installed, and remind the user that Claude Code auto-discovers skills from `~/.claude/skills/` (run `/skills` or restart to see them). Surface any prerequisites noted in `README.md`.

## Rules
- **Plan first, always.** Never install before the user approves the plan.
- **Never install or overwrite a skill the user didn't explicitly select.** Flag overwrites in the plan.
- No secrets, customer data, or tenant IDs live in this repo; keep it that way.
- This repo contains only skill folders + docs — there is nothing to build, compile, or run.

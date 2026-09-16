# Setup instructions for Claude

You are reading this because the user pointed you at this repository. Your job is to help them **install the skills they want** from it. Follow this protocol exactly — do not improvise an install.

## When the user asks to set up / install / add these skills (or just "set this up")

1. **Enter plan mode immediately.** Do **not** copy, write, register, or modify anything until the user approves a plan. Setup is a plan-first operation.
2. **Build the catalog.** Find every `SKILL.md` under `skills/` (they may be nested in category folders). For each, show its folder name and the one-line `description:` from its frontmatter. Group by category folder if present.
3. **Offer two selection modes and ask the user which they want:**
   - **Prescriptive** — they pick specific skills by name or number (accept lists and ranges), **or**
   - **Select all** — install every skill.
   If `skills/references/` exists, treat it as shared dependency files (e.g. `dt-brand` needs them) — include it automatically whenever a skill that depends on it is selected, or with "all".
4. **Classify the selection into two install targets** (same layout as the README "Installation" section):
   - **Dependency skills → `~/.agents/skills/<name>/`, symlinked into the skills dir.** These are the bundled dynatrace-for-ai copies: `dt-dql-essentials` and every `dt-obs-*` (use the bundled `dt-obs-frontends`, not upstream's — `dt-dem-vitals`/`dt-rum-journey` need its CamelCase reference files), plus the `dtctl` skill, installed with `dtctl skills install --cross-client --global` (lands in `~/.agents/skills/dtctl`).
   - **Package skills → the skills dir** (`~/.claude/skills/<name>/`): everything else under `skills/`.
   - Nine skills (`dt-ai-obs`, `dt-cloud-cost`, `dt-dem-vitals`, `dt-deploy-risk`, `dt-k8s-podf`, `dt-rcf`, `dt-rum-journey`, `dt-slo-burn`, `dt-vuln-blast`) read hard-coded `~/.agents/skills/<skill>/...` paths, including `~/.agents/skills/dtctl/references/DQL-reference.md`. If any of them is selected, add all dependency skills and the `dtctl` skill to the plan automatically.
5. **Present the plan via ExitPlanMode.** Spell out: the exact skills to install and each one's target (`~/.agents/skills/<name>/` + symlink, or `~/.claude/skills/<name>/`), the `dtctl skills install --cross-client --global` step if included, and **explicitly flag any skill or symlink that already exists and would be overwritten** (including a non-symlink copy in the skills dir where a symlink will go).
6. **Only after approval, apply it** (create target dirs if missing; never touch skills the user didn't choose):
   ```bash
   SKILLS_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}/skills"
   mkdir -p "$HOME/.agents/skills" "$SKILLS_DIR"
   # dependency skill <dep> (from skills/):
   cp -R "skills/<dep>" "$HOME/.agents/skills/"
   ln -sfn "$HOME/.agents/skills/<dep>" "$SKILLS_DIR/<dep>"
   # dtctl skill:
   dtctl skills install --cross-client --global
   ln -sfn "$HOME/.agents/skills/dtctl" "$SKILLS_DIR/dtctl"
   # package skill <name>:
   cp -R "skills/<name>" "$SKILLS_DIR/"
   ```
   Honoring `$CLAUDE_CONFIG_DIR/skills` changes only where package skills and symlinks go. The `~/.agents/skills/` dependency layout is required no matter where the config dir is — the nine skills above read those absolute paths.
7. **Report** what was installed, verify `~/.agents/skills/dtctl/references/DQL-reference.md` and `~/.agents/skills/dt-obs-frontends/references/WebVitals.md` exist when dependencies were installed, and remind the user that Claude Code auto-discovers skills from the skills dir (run `/skills` or restart to see them). Surface any prerequisites noted in `README.md`.

## Rules
- **Plan first, always.** Never install before the user approves the plan.
- **Never install or overwrite a skill the user didn't explicitly select** (required dependencies added in step 4 count only once shown in the approved plan). Flag overwrites in the plan.
- No secrets, customer data, or tenant IDs live in this repo; keep it that way.
- This repo contains only skill folders + docs — there is nothing to build, compile, or run.

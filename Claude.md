# Local Claude Code Development Control Plane

## 1. Simple Bootstrap Prompt

Copy this into Claude Code from the root of a repo:

```text
Set up this repository for a local-only Claude Code workflow.

Goal:
Reduce human context-management burden when working across Java, C++, JavaScript, TypeScript, Python, and Android projects.

Important constraints:
- Use local-only MCP servers where possible.
- Make Serena the primary MCP for semantic code navigation.
- Use CLI-based skills for build/test/debug workflows instead of adding many MCPs.
- Do not configure remote SaaS MCPs.
- Do not store secrets, tokens, API keys, or credentials in repo files.
- Do not modify application code during this setup.

Create or update this structure:

CLAUDE.md
docs/current-focus.md
docs/context-map.md
docs/project-log.md
docs/decisions/ADR-000-template.md
docs/agent-handoffs/handoff-template.md

.claude/agents/context-librarian.md
.claude/agents/code-reviewer.md
.claude/agents/test-runner.md
.claude/agents/mcp-auditor.md

.claude/skills/init-local-repo/SKILL.md
.claude/skills/finish-task/SKILL.md
.claude/skills/handoff/SKILL.md
.claude/skills/serena-first-code-nav/SKILL.md
.claude/skills/java-gradle/SKILL.md
.claude/skills/android-validate/SKILL.md
.claude/skills/maestro-flow/SKILL.md
.claude/skills/cpp-cmake/SKILL.md
.claude/skills/js-ts-validate/SKILL.md
.claude/skills/python-validate/SKILL.md
.claude/skills/gitlab-cli/SKILL.md

bootstrap/bootstrap-local-ai-tools.sh
bootstrap/verify-local-tools.sh
bootstrap/install-npm-tools.sh

Also:
- Add .ai-tools/ to .gitignore.
- Add reports/ directories for validation output.
- Detect whether this repo uses Gradle, Android, CMake, Node/pnpm/npm, Python/uv, and GitLab.
- Configure Serena as a local-scope MCP if Claude Code and Serena are installed.
- Write a setup handoff under docs/agent-handoffs/YYYY-MM-DD-init-local-repo.md.

For the final response, explain:
1. What files were created.
2. What local tools are expected.
3. What MCPs are expected.
4. What still needs manual installation.
5. How the team should use this workflow day to day.
```

---

## 2. Coworker Explainer

## Purpose

Our team is starting to use Claude Code as a local development assistant across polyglot repositories: Java, C++, JavaScript, TypeScript, Python, and Android. The goal is not to let AI run freely or replace our engineering process. The goal is to reduce the amount of context each engineer must personally remember while improving consistency in how we inspect code, run tests, validate applications, and hand off work.

The problem this solves is simple: when multiple AI-assisted coding sessions happen across multiple repositories, the human becomes the “context router.” We end up remembering which repo had which issue, what commands were run, what Claude already tried, what failed, and what still needs review. That does not scale.

This proposal creates a lightweight local control plane around Claude Code. Each repository gets a standard set of project memory files, local skills, local validation scripts, and optional local MCP servers.

## Design Principles

The workflow is built around four principles.

First, **project context should live in the repo, not in chat history**. Every repo should have `CLAUDE.md`, `docs/current-focus.md`, `docs/context-map.md`, `docs/project-log.md`, `docs/decisions/`, and `docs/agent-handoffs/`. These files make the repo understandable to future Claude sessions and future humans.

Second, **use one primary local MCP for code intelligence**. Serena should be the standard MCP because it provides local semantic code navigation and integrates directly with Claude Code.

Third, **use skills for repeatable CLI workflows**. Claude Code skills are designed for repeated instructions, checklists, and multi-step procedures. A skill is a `SKILL.md` file with frontmatter and instructions, and it can be invoked directly with `/skill-name`.

Fourth, **prefer CLI skills over broad MCPs for build/test automation**. For example, Gradle, adb, CMake, pnpm, pytest, ruff, and glab should be wrapped in skills. This gives Claude a safe, repeatable workflow without exposing unnecessary tool surfaces.

## Standard Repository Layout

Every repository should get the following structure:

```text
CLAUDE.md
docs/
  current-focus.md
  context-map.md
  project-log.md
  decisions/
    ADR-000-template.md
  agent-handoffs/
    handoff-template.md

.claude/
  agents/
    context-librarian.md
    code-reviewer.md
    test-runner.md
    mcp-auditor.md
  skills/
    init-local-repo/
      SKILL.md
    finish-task/
      SKILL.md
    handoff/
      SKILL.md
    serena-first-code-nav/
      SKILL.md
    java-gradle/
      SKILL.md
    android-validate/
      SKILL.md
    maestro-flow/
      SKILL.md
    cpp-cmake/
      SKILL.md
    js-ts-validate/
      SKILL.md
    python-validate/
      SKILL.md
    gitlab-cli/
      SKILL.md

bootstrap/
  bootstrap-local-ai-tools.sh
  verify-local-tools.sh
  install-npm-tools.sh

reports/
  android/
  web/
  python/
  cpp/
  maestro/
```

The important point is that this is not “AI bureaucracy.” These files let Claude Code rehydrate the project state at the beginning of a session and leave behind a clean handoff at the end.

## Local MCP Strategy

The default local MCP baseline should be small.

```text
Required:
- Serena

Optional:
- Playwright MCP for exploratory browser validation
- glab MCP only if the team accepts the risk
```

Serena should be the standard because it gives Claude Code semantic awareness of the codebase without relying on remote services. It should be configured locally per developer or per project. A typical command is:

```bash
claude mcp add --scope local serena -- serena start-mcp-server --context claude-code --project "$(pwd)"
```

For browser testing, Playwright should primarily be used through the CLI and project tests. The Playwright MCP server is useful for exploratory browser interaction, but for repeatable validation, the preferred path is still package scripts such as:

```bash
pnpm test:e2e
pnpm test:a11y
pnpm validate:lighthouse
```

For GitLab, the default should be a `gitlab-cli` skill around `glab`, not an always-on MCP. That keeps issue, MR, and pipeline workflows explicit and easier to review.

## Standard Skills

The repository should include skills for the workflows we repeat often:

```text
/init-local-repo
/finish-task
/handoff
/serena-first-code-nav
/java-gradle
/android-validate
/maestro-flow
/cpp-cmake
/js-ts-validate
/python-validate
/gitlab-cli
```

The most important workflow is `/finish-task`. It should inspect the diff, run relevant validation, invoke the code reviewer, update `docs/current-focus.md`, and create a handoff file. This ensures every Claude session leaves the repo easier to understand than it found it.

The second most important workflow is `/serena-first-code-nav`. Claude should use Serena before opening large files or doing broad searches. That reduces context bloat and improves code-navigation quality.

## Local Validation by Stack

For Java and Android, Claude should prefer the Gradle wrapper:

```bash
./gradlew build
./gradlew test
./gradlew assembleDebug
./gradlew connectedDebugAndroidTest
```

For Android devices and emulators, Claude should use adb:

```bash
adb devices
adb logcat -d > reports/android/logcat.txt
adb exec-out screencap -p > reports/android/screenshot.png
```

For C/C++:

```bash
cmake -S . -B build
cmake --build build
ctest --test-dir build --output-on-failure
clang-format --dry-run
clang-tidy
```

For JavaScript and TypeScript:

```bash
pnpm typecheck
pnpm lint
pnpm test
pnpm build
```

For Python:

```bash
uv sync
uv run ruff check .
uv run pytest
uv run mypy .
```

These commands should not be hardcoded blindly. The skills should inspect the repo first, detect available scripts, and then run the smallest useful validation.

## Human Control and Safety

This system is intentionally local-first. The default setup should not connect to remote SaaS MCP servers or production systems. Claude should not run destructive actions unless explicitly approved. Examples include database migrations, production deploys, deleting branches, merging MRs, uninstalling Android apps, clearing app data, or running payment/booking flows against real accounts.

Every skill should report:

```text
What changed
Commands run
Validation result
Files touched
Risks
Next action
```

The human should only be interrupted for high-value decisions: architecture, security, production changes, credentials, payments, database migrations, and unclear product behavior.

## Recommended Rollout

Start with one repository. Add the standard files, Serena, and the core skills. Do not attempt to automate everything on day one.

The first milestone is:

```text
Claude can enter the repo, understand the current state, make a small change, validate it, and write a handoff without the human remembering all the details.
```

The second milestone is:

```text
Every repo has the same structure, so engineers and Claude sessions can move between projects without relearning the process.
```

The third milestone is:

```text
We can open a repository and ask: “What is the current state, what is blocked, and what is the next safest task?”
```

That is the practical value of this setup. It turns Claude Code from a one-off assistant into a repeatable local engineering workflow.

## Summary

This configuration gives us a lightweight AI-assisted development standard:

```text
Serena for local code intelligence.
Skills for repeatable build/test/debug workflows.
Repo docs for durable project memory.
Handoffs for session continuity.
Bootstrap scripts for consistency.
Human approval for risky actions.
```

The result is not less engineering discipline. It is more discipline, with less mental overhead.

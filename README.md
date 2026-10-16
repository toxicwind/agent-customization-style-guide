<div align="center">

# agent-customization-style-guide

**A style guide for writing correct AGENTS.md, .prompt.md, .agent.md and .instructions.md files.**

*Synthesised for the assisted-by-AI era. Rules carry flat R-NNN ids so they can be cited in code review, commit messages, and PR replies. Renumbering is forbidden.*

</div>

---

1. [Scope](#scope)
2. [Rules](#rules)
3. [Gates](#gates)
4. [Contributing](CONTRIBUTING.md)

---

## Scope

This guide governs the agent-customization file family as of October 2026:

| File | Convention | Source of truth |
|---|---|---|
| `AGENTS.md` | openai/agents.md; adopted by VS Code, Claude Code, Codex | [agents.md](https://agents.md) |
| `.prompt.md` | VS Code prompt files (slash commands), `.github/prompts/` | VS Code docs, agent-customization |
| `.agent.md` | VS Code custom agents, `.github/agents/` (workspace) and `~/.copilot/agents/` (user, Copilot harness) | VS Code docs, custom-agents |
| `.instructions.md` | VS Code instructions, `.github/instructions/`, scoped by `applyTo` globs | VS Code docs, custom-instructions |
| `SKILL.md` | agent skills format | estate grasp-hub convention |

Out of scope: harness behavior itself, model selection, MCP server config.

## Rules

### R-1xx: AGENTS.md

**R-101.** An `AGENTS.md` states at the top which file it refines and asserts "refines, never contradicts".

*Why.* Agent READMEs are resolved nearest-first down a directory tree (2,303+ context files indexed by the convention). Without an explicit refinement link, two nested files silently contradict and the agent's behavior becomes undefined. (Provenance: agents.md convention; estate practice 2026-10-16.)

Correct:
```markdown
# AGENTS.md -- component contract
> **Refines:** `/home/toxic/AGENTS.md` (the universal contract).
```

Avoid:
```markdown
# Component rules
Some rules here, no parent link, maybe contradicting the root.
```

**R-102.** Keep the root `AGENTS.md` shallow and curated; put domain detail in nested files that own their directory.

*Why.* Context is findable/scoped/pruned, never wrong/missing/stale ("Developer as Curator", ACM 2026). A wall-file drifts; nested refinement keeps each rule near the code it governs. (Provenance: estate-original.)

**R-103.** `AGENTS.md` never contains secrets, credentials, or machine-specific absolute paths that break on other hosts.

*Why.* AGENTS.md is committed and roams via git (and is a first-class prompt-injection surface read by every agent). A leaked key in AGENTS.md propagates to every clone; a hardcoded `/home/toxic/...` path breaks portability (grasp-hub's zero-`/mnt` invariant). (Provenance: estate-original, grasp-hub invariant.)

Correct:
```markdown
Secrets come from the keyring (`ranch/secretsmith`), never from this file.
```

Avoid:
```markdown
Use the API key sk-... from ~/.secrets when calling the endpoint.
```

### R-2xx: .prompt.md

**R-201.** Every `.prompt.md` begins with YAML frontmatter declaring at minimum `description`, and pins `agent:` when the prompt must not run as the default agent.

*Why.* VS Code lists prompt files by description; without it the slash command shows an empty summary in the picker. Pinning `agent: plan` makes a spec-writing prompt structurally unable to edit code (synthesis/release separation, SDAD arXiv 2608.20341). (Provenance: VS Code docs prompt-files; factoryengineering.dev Copilot conventions.)

Correct:
```markdown
---
agent: plan
description: Draft a feature spec before any code is written
---
```

Avoid:
```markdown
(no frontmatter; prompt may freely edit, purpose unclear in picker)
Write the spec...
```

**R-202.** The file extension is exactly `.prompt.md` (not `.md`) and lives under `.github/prompts/` (workspace) for team-shared commands.

*Why.* The locator only picks up `.prompt.md` files; a bare `.md` in `.github/prompts/` is silently invisible as a slash command. (Provenance: VS Code docs; factoryengineering.dev: "Copilot expects .prompt.md files... the same .claude/commands/*.md files cannot be used directly via symlink".)

**R-203.** A prompt file that orchestrates a phase of a workflow states its input contract and its definition-of-done, and never both specifies and implements in one file.

*Why.* Conflating elicitation and synthesis propagates early misinterpretations (SpecFirst, arXiv 2607.27167). Separate `spec.prompt.md` from `task.prompt.md`; verification gets its own `verifySpec.prompt.md`. (Provenance: estate SDD trio, bash-style-guide/agent-customization-style-guide.)

### R-3xx: .agent.md

**R-301.** Frontmatter is valid YAML: quote any string value containing `: ` (colon+space), including descriptions with "usage: quick, medium, thorough".

*Why.* Unquoted colons are the #1 silent YAML parse failure; VS Code drops the agent with no visible error (observed: an Explore.agent.md with an unquoted description failed to load). Validate with a YAML parser before committing. (Provenance: estate-original, incident 2026-10-16.)

Correct:
```yaml
description: 'Fast read-only exploration. Specify thoroughness: quick, medium, thorough.'
```

Avoid:
```yaml
description: Fast read-only exploration. Specify thoroughness: quick, medium, thorough.
```

**R-302.** Frontmatter contains no body prose: everything after the closing `---` is the agent prompt; nothing but YAML fields before it.

*Why.* Body sections pasted inside the frontmatter block break YAML parsing and are silently dropped (observed: a "## FFs" section embedded between `tools:` and `handoffs:` corrupted an agent). (Provenance: estate-original, incident 2026-10-16.)

**R-303.** Do not duplicate the agent's body spec inside its `handoffs:` prompt; reference the file instead.

*Why.* Duplication doubles context cost on every load and guarantees drift -- the handoff copy edits while the body does not. Observed: a 1.6k-word compressed spec in a handoff prompt duplicated the full body spec. (Provenance: estate-original, incident 2026-10-16.)

**R-304.** User-level agents live in `~/.copilot/agents/` (Copilot harness). For Settings Sync roaming, place a byte-identical copy in the profile `prompts` folder and treat the prompts copy as sync transport only.

*Why.* `~/.copilot/agents/` is NOT included in Settings Sync (VS Code docs, migrate-customizations). The Prompts sync resource does sync `.agent.md` -- but only from the profile prompts folder. The two locations are read by different harnesses, so byte-identical copies do not create duplicate agents. (Provenance: VS Code docs + src/vs/platform/userDataSync/common/promptsSync/promptsSync.ts; estate finding 2026-10-16.)

### R-4xx: .instructions.md

**R-401.** Path-scoped instructions carry an `applyTo` glob in frontmatter; instructions without a scope are always-on and must justify their always-on cost.

*Why.* Always-on instructions tax every conversation's context. `applyTo: "auth/**"` auto-triggers only in matching paths (agentic-primitives pattern, github.blog 2025-10-13). (Provenance: VS Code docs custom-instructions.)

Correct:
```markdown
---
applyTo: "auth/**"
---
```

Avoid:
```markdown
(auth-specific rules with no applyTo, taxing every unrelated chat)
```

### R-5xx: Cross-cutting

**R-501.** All tracked text files are LF-only and ASCII-only.

*Why.* The main estate host is Arch Linux; CRLF from Windows authoring breaks shebangs (`env: 'bash\r': No such file or directory`) and poisons shell tooling. Observed twice on 2026-10-16. The gate flags CR bytes with the one-liner fix. (Provenance: estate-original.)

**R-502.** Every agent-customization file committed to a repo must pass a machine gate; a file that cannot be validated by tooling must not be load-bearing.

*Why.* Broken frontmatter fails silently at load time. `bin/validate.js` in this repo parses frontmatter, checks ASCII/LF, and verifies the SDD trio exists -- the TASK phase proof command. (Provenance: estate-original; grasp-hub validator pattern.)

**R-503.** Documentation references sibling repos by their local path (`ranch/<name>`), never by a hardcoded `github.com` URL. Resolve to a URL only at runtime: `git -C <path> remote get-url origin`. Within one repo, use relative links.

*Why.* Hardcoded cross-repo URLs rot on rename/migrate and duplicate what git already knows (each checkout's origin is the single source of truth -- the ranch `.gitmodules` doctrine). Relative links work on GitHub, in VS Code, and in every agent harness; runtime derivation survives repo renames for free. The only acceptable hardcoded URL is a repo referencing itself. (Provenance: estate-original; ranch .gitmodules header comment.)

## Gates

Every change must pass all gates before merge. Run from repo root:

```bash
node bin/validate.js        # convention gate: frontmatter, ASCII/LF, SDD trio
bash bin/r503-forge.sh .    # R-503 autonomous scan (pattern-forge audit / ast-grep)
```

`bin/validate.js` is the canonical proof command of the TASK phase
(`.github/prompts/task.prompt.md`). A change is not done until both gates exit 0.

**R-503 autonomous enforcement** (`bin/r503-forge.sh`): runs `forge audit
--rule bin/sg/r503.<lang>.yml` (pattern-forge's ast-grep verifier) per language
-- ts/js, python, bash, json, yaml -- against the repo tree. Any hardcoded
`github.com/<owner>/<repo>` URL in code or config is reported as evidence
(file:line + text) and the gate exits 1. Not scanned: `.md` (self-links are
sanctioned by R-503's own-reference exception) and `.gitmodules` (the
sanctioned registry).

## SDD workflow

This repo dogfoods spec-driven development:

1. `/spec` -- elicit a behavioral spec into `specs/NNN-<slug>/spec.md` (Plan agent, read-only).
2. `/task` -- synthesize the rule + gate, one proof-gated step at a time.
3. `/verifySpec` -- independent verification: Spec Fidelity + gate reproduction.

Synthesis and release authority are separated (SDAD, arXiv 2608.20341).
Elicitation precedes synthesis (SpecFirst, arXiv 2607.27167).

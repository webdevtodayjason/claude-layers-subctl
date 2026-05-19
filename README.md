# claude-layers

[![Latest release](https://img.shields.io/github/v/release/frontier-infra/claude-layers?label=release&color=7C3AED)](https://github.com/frontier-infra/claude-layers/releases)
[![License: MIT](https://img.shields.io/github/license/frontier-infra/claude-layers?color=blue)](LICENSE)
[![Built for Claude Code](https://img.shields.io/badge/built%20for-Claude%20Code-D97757)](https://claude.com/claude-code)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen)](CONTRIBUTING.md)

**A layered behavioral configuration system for Claude Code.**

`claude-layers` installs a four-layer instruction architecture into your Claude Code project. The layers compose at runtime so that universal coding discipline, role-specific behavior, project-specific values, and per-task constraints all stack into a single coherent context, with clear precedence rules and no silent contradictions.

If you have ever watched a Claude Code agent silently rewrite a working file, capitulate the moment you challenged it, or invent requirements that were not in the spec, this is the tool for those problems.

```bash
curl -fsSL https://raw.githubusercontent.com/frontier-infra/claude-layers/main/install.sh | bash
```

Or clone and run manually:

```bash
git clone https://github.com/frontier-infra/claude-layers.git
cd claude-layers
./install.sh /path/to/your/project
```

---

## Why this exists

In late 2025, the bar for what an AI coding agent could do crossed a threshold. Andrej Karpathy summarized the failure modes that remain ([source](https://x.com/karpathy/status/2015883857489522876), [HN discussion](https://news.ycombinator.com/item?id=46771564)):

> They make wrong assumptions on your behalf and just run along with them without checking. They don't manage their confusion, they don't seek clarifications, they don't surface inconsistencies, they don't present tradeoffs, they don't push back when they should, and they are still a little too sycophantic... They also really like to overcomplicate code and APIs, they bloat abstractions, they don't clean up dead code after themselves.

A single `CLAUDE.md` at the root of your project addresses some of this. But a single flat file struggles when:

- You work on multiple projects with different values, vocabularies, and constraints
- You want specialized agents (builder, writer, investigator) with clear ownership boundaries
- You want the agent to know when to escalate to you versus when to act
- You want the same agent to behave differently depending on which project it is in

`claude-layers` solves these by separating concerns into composable layers.

## The layering model

```
Layer 4: Task context        (what you typed, right now)
Layer 3: Project overlay     (.claude/projects/PROJECT_*.md)
Layer 2: Role overlay        (.claude/agents/*.md)
Layer 1: Base CLAUDE.md      (project root)
```

**Lower-numbered layers take precedence.** Higher layers add constraint, never relax it.

Concretely: if your project overlay says "Forge may proactively refactor adjacent code" but the base `CLAUDE.md` says "do not improve adjacent code as a side effect," the base wins. The project overlay is misleading and gets surfaced as a conflict, not silently applied.

This precedence model is the whole point. It means you can write project overlays without worrying about accidentally undermining the universal discipline, and you can write role overlays without worrying about being overridden by overzealous project instructions.

## What gets installed

```
your-project/
├── CLAUDE.md                              # Layer 1: universal discipline
└── .claude/
    ├── agents/
    │   ├── FORGE.md                       # Layer 2: builder role
    │   ├── QUILL.md                       # Layer 2: writer role
    │   ├── SCOUT.md                       # Layer 2: investigator role
    │   └── WARDEN.md                      # Layer 2: verifier role (/goal sign-off)
    ├── commands/
    │   ├── goal.md                        # /goal: open a verifiable contract
    │   └── goal-verify.md                 # /goal-verify: run Warden, sign off
    ├── goals/                             # /goal manifests + Warden proof artifacts
    ├── projects/
    │   ├── PROJECT_TEMPLATE.md            # blank template for new projects
    │   ├── PROJECT_WEBAPP.md              # example: full-stack web app
    │   ├── PROJECT_API.md                 # example: REST/GraphQL service
    │   └── PROJECT_CLI.md                 # example: command-line tool
    ├── orchestrator/
    │   └── ARGENT.md                      # routing and context composition
    ├── docs/
    │   ├── LAYERING_MODEL.md              # how layers compose at runtime
    │   ├── ROUTING_PATTERNS.md            # when to use which agent
    │   ├── CREATING_OVERLAYS.md           # how to write a new overlay
    │   └── GOAL_PROTOCOL.md               # /goal manifest + Warden + Stop hook
    └── settings.example.json              # reference Stop-hook config for /goal gating
```

The example projects (`WEBAPP`, `API`, `CLI`) are reference material. Delete the ones you do not need and copy `PROJECT_TEMPLATE.md` to create your own.

## How it works at runtime

Claude Code reads `CLAUDE.md` at the project root automatically. The `.claude/` subdirectory holds the rest of the layers, which you reference explicitly when invoking an agent.

A typical invocation:

```
Forge: read FORGE.md and PROJECT_WEBAPP.md, then add an idempotency key to the
webhook handler in src/api/stripe.ts.
```

What happens in Claude's context window, in order:

1. `CLAUDE.md` is already loaded (Claude Code does this automatically)
2. `FORGE.md` is loaded (you asked for it by name)
3. `PROJECT_WEBAPP.md` is loaded (you asked for it by name)
4. The task itself is in your message

The agent now has:

- Universal discipline (read before write, no scope creep, hold position under challenge)
- Builder role discipline (you implement, you do not architect; tests before code)
- Project-specific rules (idempotency keys are mandatory on webhook handlers in this project)
- A concrete task

The agent has enough context to do the work correctly and enough boundaries to refuse if the task implies overstepping.

## The four roles

`claude-layers` ships with four roles. The pattern is deliberate: they cover the full lifecycle of any technical change — *what exists* (Scout), *what to do* (Quill), *the doing* (Forge), and *did it actually meet the contract* (Warden).

| Role | What they do | What they refuse |
|---|---|---|
| **Forge** | Implements code against a spec | Decides what to build, expands scope, makes architecture decisions |
| **Quill** | Writes specs, docs, READMEs, comments | Writes code (except illustrative snippets) |
| **Scout** | Investigates existing systems, traces flows, reproduces bugs | Decides what to change, writes the fix |
| **Warden** | Verifies that delivered work meets its `/goal` manifest, writes the proof artifact, signs off or kicks back | Patches failures, edits the manifest, negotiates partial sign-off |

This separation is the source of most of the value. When an agent has a narrow role with explicit "what you do not own" boundaries, it stops doing the things you did not ask for. When you want it to do something outside its role, you invoke a different role.

The "what you do not own" sections in each overlay are deliberately heavier than the "what you own" sections. That asymmetry is intentional: bounding behavior is harder than enabling it, and most LLM coding failures are scope-creep failures.

## The `/goal` contract (v1.1.0+)

Every code-changing slice gets a manifest at `.claude/goals/<task-id>.yaml` that declares its `done_when` checks (tests, commands, file constraints, diff constraints). Workers cannot self-assess "done" — Warden runs the manifest, writes `.claude/goals/<task-id>.proof.json`, and a Stop hook refuses to release the worker until `signed_off: true`. For Scout investigations and design slices where the verifier is operator judgment, Warden emits `signed_off: "pending"` plus a reviewer checklist; the operator flips it manually.

Full spec lives at `.claude/docs/GOAL_PROTOCOL.md`. Reference Stop-hook config at `.claude/settings.example.json`.

## The orchestrator (optional but useful)

`ARGENT.md` is the orchestrator overlay. You can use it as the configuration for a top-level agent that routes requests to specialists, or you can read it as a guide for how *you* should manually route work to the right role.

For solo developers, the orchestrator is mostly a thinking aid. You read `ROUTING_PATTERNS.md`, internalize when to use Scout versus Forge versus Quill, and the orchestrator config becomes the reference doc you point new agents at.

For more complex setups (multi-agent systems, custom Claude integrations), `ARGENT.md` is the literal config you load into your orchestration layer.

## Project overlays

Project overlays encode the things that are true for one project specifically:

- Non-negotiable rules (security boundaries, data-handling requirements)
- Vocabulary (the exact terms your project uses)
- Engineering constraints (test requirements, library choices, architectural invariants)
- Active scope (what is being worked on, what is frozen)
- Role-specific notes (what each role needs to know in this project)

The included examples (`PROJECT_WEBAPP.md`, `PROJECT_API.md`, `PROJECT_CLI.md`) demonstrate the pattern across three common project shapes. They are not your project. They are reference material showing how to structure your own overlays. Copy `PROJECT_TEMPLATE.md`, fill it in for your project, and delete the examples.

## What this does not do

To be honest about scope:

- **It does not enforce the rules.** Claude Code follows the instructions, but a sufficiently determined model can still violate them. The layers raise the floor; they do not guarantee compliance.
- **It does not eliminate the need for review.** Karpathy's "watch them like a hawk" advice still applies. This system makes the hawk's job easier, not unnecessary.
- **It does not replace project-specific knowledge.** A great project overlay still requires you to know what matters in your project. The framework is the scaffolding; you supply the content.
- **It does not auto-detect project context.** When you have multiple project overlays, you point Claude at the right one explicitly. Future versions may include path-based auto-loading.

## The failure modes this addresses

Each layer attacks a subset of common LLM coding failures:

| Failure mode | Where it is addressed |
|---|---|
| Wrong assumptions run with silently | `CLAUDE.md` section 2 (surface confusion) |
| Sycophantic capitulation | `CLAUDE.md` section 3 (hold position) |
| Bloated abstractions | `CLAUDE.md` section 4 (simplicity default) |
| Scope creep into adjacent code | `CLAUDE.md` section 5 (surgical edits) |
| Comments and code edited as side effects | `CLAUDE.md` section 7 |
| Reporting intent instead of result | `CLAUDE.md` section 8 |
| Builder making architecture decisions | `FORGE.md` (what you do not own) |
| Writer drifting into implementation | `QUILL.md` (what you do not own) |
| Investigator confabulating findings | `SCOUT.md` (citation discipline) |
| Worker self-assessing "done" without verification | `WARDEN.md` + `/goal` manifest + Stop hook |
| Orchestrator dispatching "general-purpose" workers | `ARGENT.md` dispatch contract (role + project + manifest required) |
| Project-specific rule violations | Project overlay non-negotiable rules |
| Wrong project context applied | Orchestrator project identification |

Read `.claude/docs/LAYERING_MODEL.md` after install for a deeper walkthrough.

## Installation

The installer is interactive. It checks before overwriting anything.

```bash
./install.sh                    # installs into current directory
./install.sh /path/to/project   # installs into specified directory
./install.sh --minimal          # base + roles + template only, skip example projects
./install.sh --yes              # non-interactive, accept all defaults
```

After install, run the smoke test in your Claude Code session:

```
Read CLAUDE.md, then summarize in three sentences what behaviors it requires.
```

A correct response will mention reading before writing, surfacing confusion before acting, and surgical edits. If the agent skips these, the file did not load correctly.

## Versioning your overlays

Each layer file has a `Version:` line at the top. Bump it when behavior changes. This matters most for project overlays, which evolve as your product evolves. Role overlays and base `CLAUDE.md` should rarely change. Check the overlays into your repo and review changes in PRs just like code.

## Customization

The defaults are opinionated but the structure is the point. Common customizations:

- **Replace the three roles** if your work needs different specialists. The pattern (identity, what-you-own, what-you-do-not-own, discipline, failure modes, handoff protocol) is what to preserve. The specific roles are illustrative.
- **Add roles** by copying an existing role overlay and editing. Common additions: a security reviewer, a performance specialist, a UX writer.
- **Tighten or loosen `CLAUDE.md`** if section 4's simplicity defaults conflict with your team's house style. Just remember that loosening propagates to every agent in every project.

## Background and design rationale

The four-layer model emerged from operating multi-agent systems in production. Single-file `CLAUDE.md` configurations work for one project but break when you cross project boundaries, and they conflate three distinct concerns: how an agent should think in general, what role it plays, and what the current project values.

Separating these into layers means:

1. You can fix universal failure modes once, in `CLAUDE.md`, and every agent in every project benefits.
2. You can specialize without rewriting the base. Adding a new role is one file.
3. You can switch projects without context contamination. The agent reads a different overlay; the rest of the stack is the same.
4. You can audit behavior by reading the layers. The reasons for any agent decision should trace back to specific layer sections.

The role separation (build / write / investigate) maps to the actual lifecycle of any change: you understand the system (Scout), you decide what to do (Quill), you do it (Forge). Conflating these is the source of most agent failures.

The "what you do not own" sections are the most important part of each role overlay. They are the explicit license to refuse work that does not fit. Without them, agents drift toward becoming generalists who are mediocre at everything.

## Contributing

This is an opinionated starting point. PRs welcome for:

- Additional example project overlays for common project shapes
- Additional roles that have proven useful in practice
- Documentation improvements
- Installer bugs

Not accepted:

- Changes that relax the precedence rules
- Changes that conflate role and project concerns
- Decoration without operational impact

## License

MIT. Use it, modify it, ship it. Attribution is appreciated but not required.

## Acknowledgments

The base `CLAUDE.md` discipline draws directly on the failure modes Andrej Karpathy documented in his late-2025 Claude Code notes ([X thread](https://x.com/karpathy/status/2015883857489522876) · [Hacker News](https://news.ycombinator.com/item?id=46771564)). The layered architecture is influenced by years of building multi-agent systems where flat configurations consistently failed at scale.

# Claude Code Control Plane

This is an architecture pattern for making AI-assisted development operationally reliable for infrastructure nerds. I'm not a developer but I have spent 20+ years working on networks, data centres and infrastructure so I'm coming at AI-assisted coding from a different perspective from people that write code for a living. 

If you're a sysadmin, network engineer or infrastructure specialist using AI coding tools and wondering why everyone else's advice feels like it's written for a different audience — that's because it is :D Hopefully this helps you unlock Claude Code as another tool like it has for me!

---

## What This Is

Claude Code is a stateless execution engine. Every session starts fresh and it has no memory of previous work, no understanding of your systems, no awareness of what happened five minutes ago. Left unmanaged it's a fast but unreliable worker that guesses at context and sometimes makes confident mistakes.

This 'control plane' is an infrastructure model that helps fix this. It connects a human operator (who has domain knowledge, architectural judgment and system understanding) to a stateless AI executor (who has speed, broad technical knowledge, and no context) and makes the combination reliable.

This isn't a guide to writing a good CLAUDE.md. The specific rules are personal to whatever operator builds them. My CLAUDE.md isn't going to make *your* projects better. This is about the **architecture pattern** — how the components fit together, what role each one plays and why it works for someone who thinks in systems rather than syntax.

## Friction

A stateless executor has fundamental operational limitations:

- **No persistence** — it doesn't remember what it did last session, what phase a project is in, or what mistakes it made before.
- **No judgment about your systems** — it knows how frameworks work generally, but not how *your* app is structured, what your data formats look like, or where your deployment targets are.
- **No guardrails** — it will introduce type errors, skip tests, deploy without version bumps, and declare itself done with broken code.
- **No consistency** — without shared policy, behavior varies between sessions and across projects. The same mistake gets made repeatedly because lessons don't persist.

I view each of these as an infrastructure problem.

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   OPERATOR                      │
│  Domain knowledge · Architectural judgment      │
│  System understanding · Acceptance criteria     │
└──────────────────────┬──────────────────────────┘
                       │
              ┌────────┴────────┐
              │  CONTROL PLANE  │
              └────────┬────────┘
              ┌────────┴───────────────────────┐
              │                                │
   ┌──────────┴──────────┐    ┌────────────────┴───────────┐
   │   Policy Layer      │    │   State Layer              │
   │                     │    │                            │
   │  Global CLAUDE.md   │    │  Context files             │
   │  Project CLAUDE.md  │    │  (per-project state store) │
   │ (behavioral policy) │    │                            │
   └──────────┬──────────┘    └────────────────┬───────────┘
              │                                │
   ┌──────────┴──────────┐    ┌────────────────┴───────────┐
   │   Validation Layer  │    │   Automation Layer         │
   │                     │    │                            │
   │  PostToolUse hooks  │    │  Skills (runbooks)         │
   │  Stop hooks         │    │  SSH agent forwarding      │
   │ (admission control) │    │  (execution reach)         │
   └──────────┬──────────┘    └────────────────┬───────────┘
              │                                │
              └────────┬───────────────────────┘
                       │
              ┌────────┴────────┐
              │    EXECUTOR     │
              │  (Claude Code)  │
              │                 │
              │  Stateless      │
              │  Fast           │
              │  No memory      │
              │  No judgment    │
              └─────────────────┘
```

## Directory Structure

The control plane lives alongside the projects it manages:

```
~/Projects/
├── context/                          # Private repo — the control plane itself
│   ├── global-claude.md              # Global policy file
│   ├── ProjectA_CLAUDE.md            # Project-level policy
│   ├── ProjectA_PROJECT_CONTEXT.md   # Project state store
│   ├── ProjectB_CLAUDE.md
│   ├── ProjectB_PROJECT_CONTEXT.md
│   └── ...
│
├── ProjectA/                         # Project repo (public or private)
│   ├── CLAUDE.md                     # ← symlink to context/ProjectA_CLAUDE.md
│   ├── .gitignore                    # includes CLAUDE.md
│   ├── src/
│   └── ...
│
├── ProjectB/
│   ├── CLAUDE.md                     # ← symlink to context/ProjectB_CLAUDE.md
│   ├── .gitignore                    # includes CLAUDE.md
│   ├── src/
│   └── ...
```

The `context/` directory is a single private Git repository. It contains every policy file, every context file, and any planning specs or design docs. This is the control plane's source of truth.

Project repos receive their CLAUDE.md via symlink from the context directory. The symlink is gitignored so it never appears in the public repo. This means:

- All policy and state files are version-controlled in one place
- Public repos don't expose your operational instructions
- Updating a policy file in `context/` immediately propagates to the project
- Claude Code reads the symlinked CLAUDE.md at session start like any other file

The naming convention (`ProjectName_CLAUDE.md`, `ProjectName_PROJECT_CONTEXT.md`) is flat and deliberate — no subdirectories, no nesting. Every file in the context repo is visible at a glance.

## Components

### Policy Engine — CLAUDE.md Files

**Infrastructure role:** Behavioral constraints propagated to every session.

A global CLAUDE.md defines rules that apply across all projects. Project-level CLAUDE.md files extend or override for specific situations. The executor reads these at session start and operates within their boundaries (ideally).

The content of these files is whatever the operator needs. The point isn't *what* the rules say — it's that **rules exist as infrastructure, not as things you remember to say each time.** A rule learned from a mistake gets encoded once and enforced forever. Without this, the same failure mode recurs across sessions because the executor has no memory of having caused it in the first place.

Properties:
- **Declarative** — describes desired behavior, not step-by-step procedures
- **Layered** — global policy applies everywhere, project policy scopes to specific workloads
- **Persistent** — survives session boundaries, unlike conversational instructions
- **Evolvable** — grows organically as failure modes are discovered and encoded

### State Store — Context Files

**Infrastructure role:** Persistent knowledge that survives session boundaries.

Each project has a context file that tracks current state: what the project is, its structure, what phase it's in and how things connect. These live in a dedicated repository, separate from the projects themselves.

This solves the fundamental statefulness problem. A stateless executor can't know that your project just shipped v0.6.0, that the homepage changed from one view to another, or that there are 43 routes instead of 33. Context files give it that knowledge without the operator re-explaining every session.

Properties:
- **Current state, not history** — change history lives in the CHANGELOG where it belongs. 
- **Source of truth** — verified against actual codebases, not left to drift
- **Maintained as infrastructure** — updated after meaningful work, like updating documentation after a build.
- **Centralized** — all context files in one repo, distributed to projects as needed

### Admission Controllers — Hooks

**Infrastructure role:** Automated validation at execution boundaries.

Hooks intercept the executor at defined lifecycle points and enforce quality gates:

- **PostToolUse hooks** fire after every file edit. A typecheck hook catches type errors and syntax errors immediately, before they compound into cascading failures. The executor gets the error fed back and must fix it before continuing.
- **Stop hooks** fire when the executor tries to declare "done." A test hook runs the project's test suite against modified files. If tests fail, the executor is blocked from stopping and must fix the failures first.

This is the same pattern as admission controllers in Kubernetes — validate at the boundary, reject what doesn't meet policy. Without these, the executor can (and will) introduce errors, ignore them and hand back broken code.

Properties:
- **Automated** — no operator intervention required to enforce
- **Blocking** — failures prevent the executor from proceeding, not just warn
- **Targeted** — different hooks for different lifecycle events, scoped to relevant file types
- **Fail-safe** — if the hook itself fails, it exits cleanly rather than blocking all work

### Runbooks — Skills

**Infrastructure role:** Codified multi-step procedures triggered on demand.

Repetitive workflows (release pipelines, deployment sequences, build procedures) get encoded as skills — reusable prompts the operator invokes with a single command. This eliminates missed steps in multi-stage operations and ensures consistency across executions.

A release skill that enforces "bump version, build both targets, update changelog, commit, push" prevents the recurring failure of deploying without a version bump — not because the operator remembers, but because the procedure is coded.

### Config Distribution — Symlinks and Gitignore

**Infrastructure role:** Shared configuration from a private source of truth.

Project CLAUDE.md files can be symlinked from a private context repo into public project repos, then gitignored. This means:
- Configuration is version-controlled in one place
- Public repos don't expose operational instructions
- Updates propagate by updating the source, not each project individually

This is the same pattern as managing configuration through a central config management system rather than editing files on individual devices.

### Network Layer — SSH Agent Forwarding

**Infrastructure role:** Extending execution reach beyond the local environment.

If the executor runs in a containerized environment (distrobox, devcontainers, etc.), SSH agent forwarding through the host gives it access to remote servers for deployment, diagnostics, and verification — without storing credentials in the container. The executor can reach a destination through a controlled path.

### Web Bridge — ClaudeLink

**Infrastructure role:** Shared state access across interfaces.

Claude Code operates from the terminal with direct filesystem access. Claude's web interface is where longer planning, design thinking, and cross-project reasoning happen. Without a bridge between them, context must be manually relayed — copy-pasting state between interfaces, re-explaining project context in web sessions, or working blind.

The bridge uses the context repository as shared state between both interfaces:

```
┌──────────────┐         ┌──────────────────┐         ┌──────────────┐
│  Claude Web  │◄───────►│  GitHub          │◄───────►│  Claude Code │
│  (planning)  │   MCP   │  (context repo)  │   git   │  (execution) │
└──────────────┘         └──────────────────┘         └──────────────┘
```

**Claude Code** reads and writes context files via the filesystem and pushes changes to GitHub through normal git operations.

**Claude Web** reads and writes the same files via a GitHub MCP connector — a remote MCP server that gives the web interface direct access to your GitHub repositories through the API.

This means a web session can read the current state of any project, update planning documents, or review what Code did in the last session — all without the operator manually relaying information. It also works from any device: phone, tablet, a Steam Deck, anything with a browser.

**Setup:** Claude's web interface supports [remote MCP server connections](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp) (Pro/Max/Team/Enterprise plans). GitHub provides an [official remote MCP server](https://github.com/github/github-mcp-server) with [configurable toolsets and access controls](https://github.com/github/github-mcp-server/blob/main/docs/remote-server.md). Connecting the two gives web sessions authenticated read/write access to your repositories, including the private context repo that forms the control plane's state store.

The web interface becomes a planning and coordination layer. Code remains the execution layer. The context repo is the shared state that connects them.

## How Sessions Work

Each Claude Code session is a **stateless worker**:

1. **Boot** — session starts, reads global CLAUDE.md and project CLAUDE.md
2. **Context load** — operator points it at relevant project and loads the context file
3. **Execute** — operator provides instructions, executor works within policy constraints, hooks validate at boundaries
4. **State update** — context files and changelogs are updated to reflect completed work
5. **Terminate** — session ends, all in-memory context is lost

The next session starts from scratch, but the control plane ensures it has everything it needs to be productive immediately. No re-explaining the project, no re-establishing rules, no re-learning from past mistakes.

This is operationally similar to how stateless application servers work behind a load balancer: any instance can handle any request because shared state lives in external stores, not in process memory.

## Why This Works

This pattern works because it maps directly to problems infrastructure engineers already know how to solve:

| Problem | Traditional Infrastructure | Control Plane |
|---|---|---|
| Stateless workers need shared state | External databases, config stores | Context files in a dedicated repo |
| Consistent behavior across instances | Policy engines, config management | Layered CLAUDE.md files |
| Catching bad output | CI/CD gates, admission webhooks | PostToolUse and Stop hooks |
| Repeatable multi-step operations | Runbooks, playbooks | Skills |
| Secure credential distribution | Secret management, agent forwarding | SSH agent socket passthrough |
| Config across environments | Central config repo, symlinks | Private context repo with symlinks |

The operator doesn't need to be a software developer to build this. They need to understand systems, failure modes, and operational discipline — which is a different skill set entirely.

## How It Evolves

The control plane wasn't designed upfront. It grew organically from operational experience:

- An auth vulnerability got missed → auth rules added to policy
- The executor kept grinding on bad approaches → two-strike debugging rule encoded
- Context files drifted from reality → maintenance rule added, verification pass scheduled
- Type errors compounded silently → typecheck hooks added as admission controllers
- Release steps got skipped → release skill codified as a runbook

Each failure mode becomes infrastructure that helps prevent recurrence. The control plane is a living system that gets more reliable over time — not because the executor gets smarter, but because the constraints around it get tighter.

This is how you operate any system reliably: not by hoping the components don't fail, but by designing the system to handle when it does.

## Getting Started

### Context Repository

Create a private Git repository for your control plane. This holds all policy files, context files, and planning documents. A flat file structure with a consistent naming convention is easier to maintain than nested directories.

### Policy Files

A global policy file defines rules that apply across all projects. Project-level policy files extend or override for specific contexts.

The content is whatever rules your projects need. Common categories include:

- **Working style** — how you want the executor to communicate, plan, and seek approval
- **Code standards** — formatting, naming, type safety, dependency management
- **Debugging rules** — what to do when a fix attempt fails, when to stop and reassess
- **Deployment procedures** — version bumping, changelog updates, build verification
- **Security constraints** — authentication rules, trust boundaries, input validation
- **Things not to do** — the mistakes that keep recurring, encoded so they stop

None of these are prescriptive. A solo hobbyist's policy file will look nothing like one managing production infrastructure. The point is that rules exist as persistent files rather than things you remember to say each session.

Policy files grow organically. Start with what you know, add rules when failures occur. A policy file that tries to anticipate everything upfront will be ignored. One that encodes real lessons from real problems gets followed.

Claude is very good at writing instructions for Claude. You don't need to write policy files from scratch — describe what you want in plain language and have Claude draft the rules. A planning session in Claude's web interface can produce the initial policy files that Claude Code then operates under. Refine them as you discover what works and what doesn't.

### Context Files

Context files are project state stores. They give a stateless executor the knowledge it needs to be productive without the operator re-explaining the project every session.

A template is provided at [`templates/PROJECT_CONTEXT.template.md`](templates/PROJECT_CONTEXT.template.md). The sections are:

- **What the project is** — one paragraph, enough for an executor starting cold to understand what it's working on
- **Environment** — how to run, build, test, and deploy. Commands, paths, prerequisites.
- **Tech stack** — frameworks, languages, key dependencies
- **Project structure** — file tree showing how the codebase is organized
- **Current state** — version, phase, what's done, what's in progress
- **Known issues and planned work** — what's broken, what's next

Context files track current state, not history. Change history belongs in git logs and changelogs. When meaningful work is completed, the context file gets updated to reflect the new state — like updating documentation after a deployment.

The template can be used as a starting point by giving it to Claude Code with instructions to fill it in based on the actual project. The executor can read the codebase and populate the sections — it's faster and more accurate than writing them manually. The operator then reviews and corrects. This is the same pattern that applies throughout: Claude writes, the operator verifies. The human provides judgment and domain knowledge; the executor provides speed and thoroughness.

### Symlinks

To distribute policy files to project repos without exposing them:

```bash
# From your project directory
ln -s ../context/ProjectName_CLAUDE.md CLAUDE.md

# Add to .gitignore
echo "CLAUDE.md" >> .gitignore
```

Claude Code follows symlinks transparently. The file appears in the project root as `CLAUDE.md` and gets read at session start like any other file.

## Background

This pattern emerged over ~225 Claude Code sessions across six concurrent projects (browser extensions, web dashboards, game development, server infrastructure) by an infrastructure specialist with 16 years in the financial technology sector — not a software developer. The framing came from realizing the workflow that had been built organically mapped directly to infrastructure patterns: policy engines, state stores, admission controllers, and runbooks. It was just invisible because it's how an infrastructure person naturally thinks about making systems reliable.

## License

[MIT](LICENSE)

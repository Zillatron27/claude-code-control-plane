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

## Background

This pattern emerged over ~225 Claude Code sessions across six concurrent projects (browser extensions, web dashboards, game development, server infrastructure) by an infrastructure specialist with 16 years in the financial technology sector — not a software developer. The framing came from realizing the workflow that had been built organically mapped directly to infrastructure patterns: policy engines, state stores, admission controllers, and runbooks. It was just invisible because it's how an infrastructure person naturally thinks about making systems reliable.

## License

[MIT](LICENSE)

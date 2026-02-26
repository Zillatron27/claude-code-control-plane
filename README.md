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

This is the same pattern as managing configuration through a central config management system rather than editing config on individual devices.

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

**Setup:** Claude's web interface supports [remote MCP server connections](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp) (Pro/Max/Team/Enterprise plans). GitHub provides an [official remote MCP server](https://github.com/github/github-mcp-server) with [configurable toolsets and access controls](https://github.com/github/github-mcp-server/blob/main/docs/remote-server.md). Connecting the two can give web sessions authenticated read/write access to your repositories, including the private context repo that forms the control plane's state store.

**Note:** This bridge depends on external services — GitHub's MCP server and Anthropic's connector feature. If either changes, the bridge may need updating. The underlying problem it solves (shared state between planning and execution interfaces) doesn't go away though. If the automated bridge breaks, or if you only have a handful of repos, copying context files manually between interfaces works fine. The MCP connector just removes that friction at scale.

The web interface is a planning and thunking layer. Code remains the execution layer. The context repo is the shared state that connects them.

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

## Case Study: DryDock

The architecture section above is abstract by design — it describes the pattern, not a specific instance. This section shows what the pattern looks like in practice by walking through a real project built using the control plane: [DryDock](https://drydock.cc), a ship blueprint cost calculator for the game Prosperous Universe.

DryDock went from empty repo to v1.0.0 in three days. It's a React + TypeScript app deployed on Cloudflare Pages that lets players design ship blueprints and price the full bill of materials across six commodity exchanges. A major community shipyard operator integrated DryDock's permalink system into his workflow within hours of launch.

I'm not a developer. I'm an infrastructure specialist. The control plane is the reason this worked.

### What the control plane provided

**Context file** — `DryDock_PROJECT_CONTEXT.md` tracked the project state across every session. When a new Claude Code session started, it didn't need me to re-explain what DryDock was, what the tech stack was, where the FIO API endpoints lived, what the formula engine did, or what had already been built. The context file carried all of that. Session 1 scaffolded the repo. Session 15 implemented permalink sharing. Both started cold and both were productive immediately because the state store gave the executor everything it needed.

The context file also encoded domain knowledge that no LLM would have on its own. Prosperous Universe ship formulas were reverse-engineered from in-game testing across 13 ships — the SSC divisor is 21, not 20 (the common community assumption was wrong). The hull plate divisor is 2.07, using a volume^(2/3) surface area approximation. The emitter algorithm is greedy cover, largest first. None of this exists in any training data. It existed in a spec file I wrote, referenced by the context file, and every session had access to it.

**Policy rules** — `global-claude.md` constrained behavior across every session. Three rules mattered most during this build:

The *two-strike debugging rule* fired multiple times. During the exchange status logic implementation, the executor's first attempt at matching APEX_'s availability calculation was wrong — it was treating "has a price" as "fully satisfiable." The fix attempt also missed the mark, conflating partial supply with unavailable supply. On the second failure, the rule kicked in: stop, explain why both approaches failed, list three fundamentally different strategies, and propose the best one. The executor came back with the correct three-tier classification (full/partial/unavailable based on supply vs. quantity needed) and got it right. Without this rule, it would have kept grinding variations of the same broken logic.

The *explicit confirmation rule* prevented premature edits. During the import/export feature, the executor explained a validation approach for blueprint JSON schemas, I said "that makes sense," and it waited. Before this rule existed, "that makes sense" would have been interpreted as "go ahead and edit six files." The rule forces the executor to distinguish between the operator understanding a plan and the operator approving execution.

The *no dead code rule* kept the codebase clean across 20+ sessions. Without it, every session would have left `// TODO: implement later` comments and commented-out blocks. Over a multi-session build, that accumulates into a codebase the operator can't navigate. The rule means each session cleans up after itself.

**Project-level CLAUDE.md** — `DryDock_CLAUDE.md` layered project-specific constraints on top of global policy. Critical ones included: formula accuracy is non-negotiable (the spec is the source of truth, if code disagrees with the spec the code is wrong), use the APEX_ design system (no Tailwind, no CSS frameworks), cache FIO API responses aggressively, and handle failures gracefully (show stale data with timestamps, don't crash). These rules meant I didn't re-state project conventions every session. They were infrastructure.

**Hooks** — TypeScript strict mode with a typecheck hook (`npx tsc --noEmit`) running after every file edit caught type errors before they compounded. In a project with ~130 lines of type definitions and strict mode enforced, a single wrong type propagates fast. The hook caught these at the boundary — the executor got the error fed back and fixed it before moving on. Without the hook, type errors would accumulate silently across multiple edits and surface as a wall of failures at build time.

**Spec-driven development** — Each major feature had a spec written collaboratively with Claude's web interface before Code touched it. The permalink spec defined the encoding scheme (12 positional digits, version-prefixed, trailing zero trimming), the import flow, the URL cleaning behavior, and the collision handling. The blueprint import/export spec defined the JSON schema, validation rules, and error messages. I described what I wanted in domain terms; the web session translated that into a technical spec; I verified the spec captured my intent.

For complex changes, Claude Code worked in Plan mode first — producing an implementation plan without editing any files. I'd then take that plan back to the web session that wrote the spec and have it validate the plan against the spec before approving execution. This is another admission controller, just human-in-the-loop via the web interface rather than automated via hooks. It caught cases where Code's plan would have drifted from the spec — not because the executor was ignoring it, but because it interpreted an ambiguous section differently than I intended. The global policy rule "do not improvise solutions when a spec already defines the approach" backed this up at the execution layer, but the plan validation step caught drift before any code was written.

**Deploy skill** — The release process (bump version in `src/version.ts`, update CHANGELOG.md, build, commit, push to trigger Cloudflare Pages deploy) was encoded as a skill. Version 0.3.1 was when this got added, after I'd already done a few manual deploys and missed steps. Every release from 0.4.0 onward used the skill. The changelog in the DryDock repo is the output — consistent format, no skipped versions, no forgotten bumps.

### What it looked like in practice

DryDock shipped 12 versions in 3 days. The changelog tells the story: initial release on Feb 22, then a rapid feature cadence — comparison tables, cherry-pick sourcing, import/export (35 tests), preset blueprints, ship stats dashboard, and permalink sharing (43 more tests) — through to v1.0.0 on Feb 24.

Each session followed the same pattern described in "How Sessions Work" above. Boot, load context, receive instructions, execute within policy, update state, terminate. The next session picked up where the last one left off because the context file had been updated, not because the executor remembered anything.

The two-strike rule fired. The typecheck hook caught errors. The deploy skill enforced process. The context file prevented re-explanation. The spec files prevented improvisation. Plan validation against specs caught drift before code was written. None of these required me to remember to do them — they were infrastructure.

The test at the end is simple: a community member with a major shipyard operation saw the launch post, opened the tool, and integrated the permalink system into his workflow the same day. The tool worked because the formula engine was accurate (spec-driven), the UX was consistent (design system policy), the sharing system was well-designed (spec-driven), and the deployment was clean (skill-driven). The control plane didn't write the code, but it made the code reliable enough to ship publicly with confidence.

### What this demonstrates

DryDock is not a complex project. It's a client-side calculator with ~2,500 lines of TypeScript. The point isn't the scale — it's that a non-developer shipped a working, tested, publicly-used tool by treating AI-assisted development as an operations problem rather than a coding problem.

The control plane components map directly to what happened:

| Component | Role in DryDock |
|---|---|
| Context file | Carried project state, domain formulas, and architecture across 20+ sessions |
| Global policy | Two-strike rule, explicit confirmation, no dead code — prevented recurring failure modes |
| Project policy | Formula accuracy, design system, API caching — project-specific constraints |
| Hooks | TypeScript strict mode typecheck after every edit — caught errors at the boundary |
| Specs | Co-authored in web sessions, plans validated against spec before execution — prevented drift on complex features |
| Deploy skill | Version bump + changelog + build + push — no missed steps after v0.3.1 |
| Symlinks | CLAUDE.md symlinked from private context repo, gitignored in public DryDock repo |

The repos are public. [DryDock](https://github.com/Zillatron27/drydock) has the code, the changelog, the spec files. The context files referenced above live in a private repo, but excerpts are shown here to illustrate the pattern. You can see the output — whether the process that produced it seems worth adopting is up to you.

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

To give a concrete sense of what policy rules look like in practice, here are some examples from my files. These aren't recommendations — they're rules I added because I hit the specific problem they solve:

- **"Two-strike rule: if a fix attempt fails, do not try a similar variation. Stop, explain why the approach failed, list at least 3 fundamentally different approaches, and propose the best one."** — Added after watching the executor burn through 15 minutes trying minor variations of the same broken approach. Without this rule it will grind on a bad idea indefinitely.
- **"Wait for explicit confirmation before making changes. Explaining the fix and getting agreement on the approach is NOT permission to edit."** — Added after the executor explained what it wanted to do, I said "that makes sense," and it immediately edited six files. Understanding a plan is not the same as approving it.
- **"No dead code, commented-out blocks, or TODO placeholders unless I ask. Clean as you go."** — Added after every session left a trail of `// TODO: implement later` comments and commented-out old code that accumulated across the codebase.
- **"Every path that issues a session cookie MUST require the user to provide a credential — not just confirm one exists server-side."** — Added after an auth vulnerability where the executor built a login flow that validated a stored API key (proving the key was valid) without requiring the user to provide it (proving they owned it).

Your rules will be different because your failure modes will be different. The starting point is to use Claude Code for a while, notice what goes wrong, and encode the fix.

Claude Code's `/insights` command can also help bootstrap this — it analyzes your usage patterns and suggests rules based on what it observes. Run it, review the suggestions, keep what's useful. More broadly: Claude is very good at writing instructions for Claude. You don't need to write policy files from scratch — describe what you want and have Claude draft the rules. A planning session in Claude's web interface can produce the initial policy files that Claude Code then operates under. Refine them as you discover what works and what doesn't.

Policy files grow organically. Start with what you know, add rules when failures occur. A policy file that tries to anticipate everything upfront will be ignored. One that encodes real lessons from real problems gets followed.

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

The template can be used as a starting point by giving it to Claude Code with instructions to fill it in based on the actual project. The executor can read the codebase and populate the sections — it's faster and more accurate than writing them manually. The operator then reviews and corrects. This is the same pattern that applies everywhere: Claude writes, the operator verifies. The human provides judgment and domain knowledge; the executor provides speed and thoroughness. 

The important thing here is you don't need to understand the syntax to know if Claude has followed your instructions. 

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

This pattern emerged over ~225 Claude Code sessions across six concurrent projects (browser extensions, web dashboards, game development, server infrastructure) by an infrastructure specialist with 20+ years in the  technology sector — not a software developer. The framing came from realizing the workflow that had been built organically mapped directly to infrastructure patterns: policy engines, state stores, admission controllers and runbooks. It was just invisible because it's how I think about designing reliable systems.

This is what works for me. It might not work for you - YMMV. 

If you're an infrastructure nerd trying to make Claude Code reliable and the developer-oriented advice isn't helping, this is somewhere to start. Take what's useful, ignore what isn't, and build the system that fits how you actually work. Happy coding! 

## License

[MIT](LICENSE)

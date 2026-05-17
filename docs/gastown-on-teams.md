# Layering Gas Town on Letta Teams

A one-page architecture pitch for evolving letta-teams into a mature
multi-agent orchestration plane by adopting the patterns Steve Yegge's
[Gas Town](https://github.com/steveyegge/gastown) iterated to over two
years, as extracted into [Gas City](https://github.com/gastownhall/gascity)'s
"five primitives + four derived mechanisms" architecture.

## Thesis

letta-teams already has the **substrate** Gas Town wishes it had:
stateful Letta agents with persistent identity, memory blocks, and
conversation forking. What letta-teams lacks are the **orchestration
patterns** Gas Town iterated to: formulas, molecules, runtime providers,
event bus, health patrol, and the layering invariants that keep all of
it composable.

Layering those patterns onto letta-teams produces something genuinely
novel — Gas Town's orchestration maturity on top of Letta's stateful
agent substrate — and avoids the failure modes of the obvious
alternatives:

- **Wholesale-adopting Gas City** drags in a Go runtime that has no
  first-class notion of a stateful agent. Gas City sessions are
  processes; Letta agents are records with memory.
- **Greenfield rebuilding** in TS without studying Gas Town first
  repeats the role-hardcoding mistakes Gas Town made before extracting
  Gas City. The MEOW abstraction took two years to discover; don't
  rediscover it.
- **Continuing ad-hoc**: tasks dispatched flat, no formulas, no
  molecule decomposition, no runtime provider seam. This is the path
  letta-teams is on by default. It works for one-off team
  orchestration but doesn't scale to reusable workflows or alternate
  agent runtimes.

## Where letta-teams already aligns

| Gas Town / Gas City             | letta-teams equivalent                  | Status |
|---------------------------------|-----------------------------------------|--------|
| Role (Mayor, Deacon, Polecat)   | Teammate with role                      | ✅ Native |
| Session                         | Conversation target (root + forks)      | ✅ Native, richer |
| Mail / Nudge                    | `runtime.tasks.dispatch` + daemon       | ✅ Native |
| Supervisor loop                 | Background daemon                       | ✅ Native |
| Council / convoy                | Agent Council                           | ✅ Native |
| Persistent identity             | Stateful Letta agent + memfs            | ✅ **Stronger than Gas Town** |

Letta's stateful agent model is a real advantage. Gas Town agents are
LLM CLI processes with no first-class memory; they reconstruct state
each turn from transcript context. Letta agents carry core memory,
archival memory, and memfs across turns. Workflows that depend on
long-running agent identity (specialist reviewers, persistent
maintainers, project-scoped agents) are cleaner here than in Gas Town.

## What's missing — the layer to add

| Gas Town / Gas City pattern                  | Missing in letta-teams         | Effort |
|----------------------------------------------|---------------------------------|--------|
| **Formulas** — TOML workflow templates       | Tasks dispatched ad-hoc         | Low |
| **Molecules** — root + child tasks w/ deps   | Tasks are flat                  | Medium |
| **Runtime provider abstraction**             | Hardcoded to Letta agents       | Low |
| **Event bus** — append-only pub/sub log      | Daemon-polled state             | Medium |
| **Health patrol** — probe + restart w/backoff | Probably per-teammate ad-hoc    | Low |
| **Layering invariants** (no upward deps etc.)| Not enforced                    | Zero |

## Layering invariants — adopt verbatim

These are the rules Gas City uses to keep the architecture from rotting.
Pin them in `AGENTS.md` and cite them in PR reviews:

1. **No upward dependencies.** Layer N never imports Layer N+1.
2. **Beads / persistent task store is the universal persistence
   substrate** for domain state. (For letta-teams today: `.lteams/` is
   already this. Make it the only source of truth.)
3. **Event bus is the universal observation substrate.** All cross-layer
   visibility goes through it.
4. **Config is the universal activation mechanism.** Features turn on
   via config presence, not hardcoded branches.
5. **Zero hardcoded roles.** If a line of TS references a specific role
   name (`backend`, `reviewer`, etc.), it's a bug. Role behavior lives
   in config and prompt templates, not code.

Gas Town accumulated two years of role-hardcoding debt before Steve
realized rule 5. letta-teams can skip the cost.

## Runtime provider seam — the highest-leverage refactor

Today `runtime.tasks.dispatch` is implicitly tied to Letta agents.
Define a `RuntimeProvider` interface (Gas City calls it
`runtime.Provider`) with five methods:

```ts
interface RuntimeProvider {
  start(spec: SessionSpec): Promise<SessionHandle>;
  stop(handle: SessionHandle): Promise<void>;
  prompt(handle: SessionHandle, content: ContentBlock[]): Promise<void>;
  nudge(handle: SessionHandle): Promise<void>;
  observe(handle: SessionHandle): AsyncIterable<SessionEvent>;
}
```

Today: one implementation — `LettaAgentProvider`. Tomorrow:
`LettaCodeSubagentProvider` (spawn letta-code workers directly),
`ACPProvider` (JSON-RPC over stdio per the
[Agent Client Protocol](https://github.com/gastownhall/gascity/blob/main/internal/runtime/acp/acp.go)),
`A2UIProvider` (mobile/web clients rendering generated UI), even a
`FakeProvider` for tests.

Cost today: ~50 lines of TS. Without it, every future runtime is a
fork of `dispatch`. With it, runtimes plug in.

## Formulas and molecules

**Formulas** are TOML workflow templates. They describe a sequence of
roles, dispatches, waits, and gates. Example:

```toml
[formula.code-review]
description = "Review a code change with a reviewer/coder/tester loop"

[[formula.code-review.steps]]
role = "reviewer"
prompt_template = "templates/review.md"
wait_for = "completion"

[[formula.code-review.steps]]
role = "coder"
prompt_template = "templates/fix.md"
depends_on = "reviewer"
wait_for = "completion"

[[formula.code-review.steps]]
role = "tester"
prompt_template = "templates/verify.md"
depends_on = "coder"
```

**Molecules** are the *instances* of formulas. A molecule is a root
task plus child tasks with dependency edges, all persisted in the
task store (`.lteams/tasks.json` becomes a flat projection of a
molecule tree).

Together, formulas + molecules turn `tasks.dispatch` from
*imperative* ("send this message to backend") into *declarative*
("run the code-review formula on this change"). The daemon walks
the molecule, dispatches steps in dependency order, retries on
failure, and presents one final result.

## Why not just adopt Gas City directly?

- **Language wall.** Gas City is Go; letta-teams is TS. Adopting the
  binary means shelling out to `gc` for orchestration, which works for
  some flows but doesn't let letta-teams own the runtime provider
  abstraction in its own type system.
- **Substrate mismatch.** Gas City's `Session` is a process. Letta's
  agent is a stateful record. Wrapping a Letta agent as a
  Gas-City-shaped session loses memory semantics and forces a
  Letta-runtime-provider in Go that has to reach back over HTTP for
  every memory operation.
- **Ownership.** letta-teams is the right place for this orchestration
  to live because its abstractions (teammate, target, council, memfs)
  already speak Letta's language. Building Gas City *into* letta-teams
  in TS gives you the maturity without the impedance.

The right relationship: letta-teams becomes "Gas Town built in TS on
stateful Letta agents." Gas City stays a reference architecture.

## Sequencing

1. **Pin the layering invariants** in `AGENTS.md`. Zero code change,
   immediate review discipline.
2. **Define `RuntimeProvider` interface**, wrap current dispatch as
   `LettaAgentProvider`. ~50 lines.
3. **Add formulas** as TOML templates parsed by the daemon. Use Gas
   City's `internal/formula/` and `internal/config/` as reference for
   schema fields. Start with one formula (`code-review`) end-to-end.
4. **Add molecules** — extend task model with `parent_task_id` and
   `depends_on`. Project today's flat `tasks.json` as a molecule of
   one. Daemon walks the molecule.
5. **Event bus** — append-only `events.jsonl` (or in-process pub/sub).
   Daemon emits, CLI / TUI consume.
6. **Health patrol** — probe teammate liveness, restart on stall with
   backoff.
7. **Role packs** — define Gas Town's role catalog (Mayor, Deacon,
   Polecat, etc.) as letta-teams teammate configs in a pack directory.
   This is "build Gas Town in letta-teams" — explicit, not implicit.

Each step is independently shippable. Stop at any point and the
result is still useful.

## Non-goals

- **Don't** rewrite letta-teams in Go. The TS ecosystem alignment with
  letta-code is a strength; preserve it.
- **Don't** make the runtime provider interface so wide that it leaks
  Letta-specific concerns. Memory blocks, conversation IDs, fork
  semantics belong above the interface, in the formula and role
  config. The interface itself stays five methods.
- **Don't** invent a new TOML schema that competes with Gas City's. If
  Gas City defines a `[formula.foo.steps]` shape, use the same field
  names. Conceptual interop has value even without binary interop.
- **Don't** fork Gas City's role names. The `Mayor` / `Deacon` /
  `Polecat` taxonomy is Gas Town's, not a universal vocabulary. Roles
  belong in packs, not in core.

## End state

A user installs `letta-teams`, runs `letta-teams daemon --start`,
points it at a formula in `./formulas/` (or a published pack), and
fires `letta-teams dispatch --formula code-review --change PR#123`.
The daemon spawns the molecule, walks dependencies, dispatches to the
right teammates (reviewer → coder → tester), routes mail between
them, recovers from stalls, and surfaces one final consolidated
result. The same architecture supports council-style deliberation,
ad-hoc one-off tasks (formula-of-one), and future runtime providers
(letta-code subagents, ACP, A2UI) without changing the orchestrator.

That's Gas Town built on Letta — stateful agents with mature
orchestration patterns, all in the Node/TS ecosystem.

## References

- Gas Town (origin): https://github.com/steveyegge/gastown
- Gas City (extracted SDK): https://github.com/gastownhall/gascity
- Gas City layering invariants: `AGENTS.md` in the Gas City repo
- Gas City formulas: `internal/formula/`
- Gas City runtime providers: `internal/runtime/`
- Gas City ACP provider (reference for transport-agnostic agent
  protocol): `internal/runtime/acp/`
- A2UI protocol (mobile/web UI rendering surface for agents):
  https://a2ui.org/, https://github.com/google/A2UI

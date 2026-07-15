# Game Architecture

Real-time and turn-based games share an architectural concern general application
design rarely centers: a **simulation** that must run correctly, repeatably, and
fast, decoupled from however it is displayed. This file covers the patterns that
follow from that — sim/presentation separation, fixed-tick update pipelines, the
command pattern for deterministic replay, determinism requirements, entity-modeling
choices (ECS vs scene-graph/node composition), game state machines and AI decision
architectures, data-driven content and procedural generation, event-driven
communication, save persistence, the netcode patterns that extend a simulation to
multiplayer, performance and testing disciplines — plus the quality attributes a
game adds to the general catalog in [`06`](06-quality-attributes-tradeoffs.md).

This file is **technology- and engine-agnostic**, consistent with the rest of
this reference: it covers the structural decisions that hold regardless of engine.
Concrete engine mechanics (node lifecycles, scripting-language specifics, engine
APIs) are deliberately out of scope here; for one engine's mapping of these
patterns, see the clearly-marked, time-sensitive
[`11-godot-engine-notes.md`](11-godot-engine-notes.md).

For the general architectural vocabulary (layering, coupling/cohesion, event-
driven architecture) this file builds on, see [`01`](01-architecture-principles.md),
[`02`](02-architecture-patterns.md), and [`03`](03-software-design-principles.md).

---

## 1. Simulation/Presentation Separation

#### Summary
Structure the game as a **pure simulation core** — deterministic state
transitions driven by explicit inputs — with a separate **presentation layer**
that observes and renders the simulation's output but never feeds back into its
decisions.

#### Problem it addresses
A simulation entangled with rendering, node/widget lifecycle, or wall-clock time
can only run inside a live, on-screen instance of the game. That blocks every
use case that needs the *same* logic to run somewhere else: a headless
offline-progress recompute, a server-side replay for anti-cheat, an automated
balance-simulation harness, or a different front-end (e.g. a companion app)
reusing the same rules.

#### Description / How it works
```
   Ordered inputs                          Event log
  (commands/actions)                     (what happened)
        │                                       │
        ▼                                       ▼
  ┌─────────────┐   state transitions   ┌───────────────┐
  │  Simulation  │ ─────────────────────▶│  Presentation │
  │    Core      │                        │    Layer      │
  │ (pure, no    │◀─── never reads back ──│ (renders,     │
  │ render deps) │      from rendering    │  animates)    │
  └─────────────┘                        └───────────────┘
```
The simulation core exposes only: (1) a way to apply an ordered input/command,
(2) the resulting state, and (3) a log of discrete events describing what
changed and why (`DamageDealt`, `StatusApplied`, `UnitDefeated` — not just a
before/after diff). Presentation subscribes to that event log and animates it;
it holds no simulation authority.

#### Benefits
- The same core runs live (rendered), headless (offline catch-up, server
  verification), and in test/balance harnesses without modification.
- Presentation bugs (a broken animation, a stuck particle system) cannot
  corrupt simulation state — the coupling is one-directional.
- Enables replay, rewind, and deterministic testing, because the core's inputs
  and outputs are explicit and serializable.

#### Costs & trade-offs
- More upfront design discipline: every "just read this render-layer value in
  the simulation for convenience" temptation must be refused.
- An event log expressive enough to drive rich presentation (not just a state
  diff) is more design work than letting presentation infer changes ad hoc.
- Two layers to keep synchronized in spirit (the event vocabulary must stay
  meaningful as the simulation grows).

#### When to use
Any game where the simulation must be trusted, reproduced, or run somewhere
other than the live rendered client — turn-based and real-time alike. Close to
mandatory once **any** of: offline/idle progress calculation, replays,
deterministic multiplayer, or automated balance testing is a requirement.

#### When not to use
A one-off game jam prototype with no reproducibility, replay, or offline-calc
requirement may reasonably skip the discipline to move faster — but retrofitting
it later is a rewrite, not a patch (see §19).

#### Decision criteria
- Does any feature require the simulation to run without rendering (offline
  calc, server verification, automated testing)? If yes, this is required, not
  optional.
- Does the simulation need to be reproduced exactly later (replay, "what would
  have happened")? If yes, the event log and the determinism requirements (§4)
  are load-bearing together.

#### Common mistakes
- Simulation code reading a rendered sprite's transform, an animation's
  progress, or any other presentation-layer value to make a gameplay decision
  — this silently couples simulation correctness to frame timing.
- Presentation re-deriving "what happened" from a before/after state diff
  instead of consuming explicit events — ambiguous when multiple effects
  resolve in the same tick (see §3, §10).
- Treating the separation as a one-time refactor rather than an enforced
  boundary — a single addition of a render-dependent shortcut months later
  reintroduces the coupling.

#### Related patterns
Hexagonal Architecture / Ports & Adapters ([`02` §3.3](02-architecture-patterns.md#33-hexagonal-architecture-ports--adapters))
— the simulation core is the "domain," presentation is a driven adapter. Command
pattern (§3). Fixed-tick pipeline (§2). Determinism requirements (§4).

#### Sources
Robert Nystrom, *Game Programming Patterns* — Update Method,
Game Loop; general Model-View separation lineage (MVC/MVP/MVVM,
[`05` §2](05-desktop-application-design.md#2-desktop-ui-architecture-patterns)).

---

## 2. Fixed-Tick Update Pipeline

#### Summary
Advance the simulation in **discrete, fixed-size time steps** independent of
the variable rendering frame rate, and let presentation interpolate between
simulation states for visual smoothness.

#### Problem it addresses
If simulation math runs directly in a variable-rate render callback, its result
depends on the machine's frame rate and on incidental timing (dropped frames,
background-app throttling). The same fight computed on a 60 Hz display and a
144 Hz display — or recomputed headlessly with no display at all — can diverge.

#### Description / How it works
```
Real time ──────────────────────────────────────────────▶
Render frames:   │ frame │  frame  │ frame │    frame    │
Fixed sim ticks:  |tick|tick|tick|tick|tick|tick|tick|tick|
```
The engine loop accumulates elapsed real time and consumes it in fixed-size
increments, running the simulation update once per increment regardless of how
often the display renders. A stalled frame doesn't skip simulation time; it
runs multiple queued ticks (bounded — see the "spiral of death" note in
Common mistakes) to catch up. Rendering reads the simulation's current (and,
optionally, previous) state and interpolates between them for smooth on-screen
motion even though the underlying state only changes at the fixed tick rate.

#### Benefits
- Simulation results are reproducible regardless of display frame rate or
  hardware speed — a prerequisite for determinism (§4).
- Decouples "how often do we simulate" from "how often do we render," so each
  can be tuned independently for correctness vs smoothness/battery.
- A stalled or dropped frame doesn't desynchronize game time from real time.

#### Costs & trade-offs
- Requires explicit interpolation/extrapolation work in the render layer to
  avoid visible stepping at low tick rates (interpolation also displays the
  simulation slightly *in the past* — usually imperceptible, but a real input-
  latency cost for twitch-sensitive games).
- An unbounded catch-up loop after a long stall ("spiral of death") can itself
  blow the frame budget — must be capped, with large gaps (e.g. resuming after
  hours away) handled as an explicit headless batch computation instead (§1),
  not by looping millions of render-interleaved ticks.

#### When to use
Any simulation whose outcome must be reproducible, tested, replayed, or
computed offline — i.e., essentially the same condition as §1.

#### When not to use
Purely cosmetic, non-simulated visual effects (ambient particle drift,
decorative animation with no gameplay consequence) can run on the variable
render tick without cost — reserve the fixed-tick discipline for anything that
affects game state or outcome.

#### Decision criteria
Does this system's output feed into game state, scoring, combat resolution, or
anything that must match across runs/machines? If yes → fixed tick. Is it
purely decorative with no state consequence? Either tick rate is fine.

#### Common mistakes
- Running gameplay-affecting logic (cooldowns, damage-over-time ticks, AI
  decisions) on the variable render callback "because it was convenient."
- Unbounded catch-up iteration after a stall, itself missing the frame budget.
- Interpolating simulation state for rendering by mutating the *actual*
  simulation state rather than a separate render-facing snapshot — this
  silently reintroduces frame-rate dependence into the "pure" sim.

#### Related patterns
Simulation/presentation separation (§1); double buffering (keep a previous and
current simulation state for interpolation without mutating authoritative
state); determinism requirements (§4).

#### Sources
Glenn Fiedler, "Fix Your Timestep!"; Robert Nystrom, *Game Programming
Patterns* — Game Loop.

---

## 3. Command Pattern for Deterministic Replay

#### Summary
Represent every player/AI decision that affects the simulation as an explicit,
serializable **command** (or "intent") object, applied to the simulation in a
defined order — rather than letting scattered code paths mutate state directly.

#### Problem it addresses
Replay, offline recomputation, and deterministic multiplayer all require
answering "given the same starting state, what sequence of decisions produced
this outcome?" That's only answerable if decisions are captured as discrete,
ordered, serializable data — not as an unrecorded trail of direct method calls
and side effects scattered across the codebase.

#### Description / How it works
```
Input source           Command                Simulation
(player, AI, script) ──▶ {type: "CastSkill",  ──▶ applied in
                          actor: unit_3,           tick order
                          skill: "Fireball",
                          target: cell(4,2)}
```
Every action a player character, an enemy AI, or a scripted automation rule
wants to take is expressed as a command: a small, serializable value describing
the *intent* (who, what, where) not the *effect*. The simulation core consumes
an ordered queue of commands per tick and applies them deterministically. A
full match/session replay is then just the recorded command list plus the
initial seed — re-running the same commands through the same simulation
reproduces the same result exactly (given the determinism requirements in §4).

#### Benefits
- A replay, a save file's "how did we get here," and a PvP verification pass
  are all the same mechanism: replay the command log.
- Commands are a natural place to validate/authorize an action (reject an
  impossible command) separately from *executing* it.
- Undo/rewind (for tooling, debugging, or a designed game mechanic) falls out
  naturally if commands are paired with their inverse or with state snapshots.

#### Costs & trade-offs
- Every mutation path must be funneled through commands — a direct "just set
  this value" shortcut anywhere breaks replayability silently.
- Command schemas need versioning discipline as the game evolves (a replay
  recorded before a balance patch must still parse, even if it no longer
  "matches" current balance).
- Slightly more indirection than directly calling a method, which can feel
  like ceremony for a small prototype.

#### When to use
Any game with replay, deterministic multiplayer, or offline-recompute
requirements (this includes automation/scripted-behavior systems, since a
scripted rule's decisions are exactly the kind of "who decided what" the
command pattern captures).

#### When not to use
A prototype with no replay/multiplayer/offline-calc surface can defer this —
but see §1's note on retrofit cost.

#### Decision criteria
Will this game need to prove/reproduce "what happened" later (replay, PvP
fairness, offline-progress fidelity, bug reports)? If yes, commands are the
concrete mechanism that makes §1 and §4 actually implementable, not just
aspirational.

#### Common mistakes
- Recording the *effect* (state diff) instead of the *intent* (command) —
  effects are derived from intent plus current state plus RNG draws; recording
  only the effect loses the ability to replay under a different starting state
  (e.g., a balance-patched simulation replaying an old match for comparison).
- No schema version on stored commands, breaking old replays after a content
  update.
- Commands that carry non-deterministic data (a wall-clock timestamp, a
  pointer/reference instead of a stable ID) — breaks reproducibility the
  moment the reference doesn't resolve identically on replay.

#### Related patterns
Event Sourcing ([`02` §6.4](02-architecture-patterns.md#64-event-sourcing)) —
the same "log of what happened, replay to derive state" idea applied to game
commands specifically; simulation/presentation separation (§1); determinism
requirements (§4).

#### Sources
Robert Nystrom, *Game Programming Patterns* — Command; Glenn Fiedler,
"Deterministic Lockstep."

---

## 4. Determinism Requirements

#### Summary
For a simulation to be **deterministic** — same initial state + same ordered
inputs + same seed → bit-identical output, every run — a set of concrete
engineering rules must hold everywhere inside the simulation kernel. Determinism
is a property of the *whole* kernel: one violation anywhere silently breaks it.

#### Problem it addresses
Replay (§3), offline recomputation (§1), lockstep/rollback multiplayer (§13),
and independently-verifiable PvP results (§12) all assume the simulation can be
re-run elsewhere/later with an identical outcome. Non-deterministic habits —
unseeded randomness, hash-order iteration, wall-clock reads, frame-rate-coupled
math — are each *individually* invisible in normal play and only surface as
diverging replays, desyncs, or "impossible" verification failures much later,
when the fix is a kernel audit, not a patch.

#### Description / How it works
The load-bearing rules:

1. **Owned, seeded randomness.** All randomness inside the simulation comes
   from explicitly-seeded PRNG streams owned by the simulation, with seeds
   recorded alongside the command log. Use separate streams per concern
   (combat rolls vs. loot vs. generation) so adding a draw in one system
   doesn't shift every subsequent draw in another. Presentation-layer
   randomness (particle jitter, cosmetic variation) uses its *own* generator,
   never the simulation's.
2. **Explicit, stable iteration order.** Never iterate hash-map/set-native
   order inside the kernel; iterate sorted keys, insertion-ordered lists, or
   explicit priority orderings. Two effects that could resolve in either order
   (e.g. a mutual kill) must have a documented resolution order.
3. **No ambient inputs.** No wall-clock time, frame delta, locale, environment
   state, or uninitialized memory feeds the simulation — only ticks (§2) and
   commands (§3).
4. **Floating-point caution.** IEEE 754 results depend on operation order,
   compiler optimizations (e.g. FMA contraction), math-library implementations,
   and platform/architecture. Same-binary-same-machine replay is achievable
   with care; **cross-platform bit-exactness with floats is hard and has no
   general solution**. Production deterministic engines commonly use integer
   or fixed-point math in the simulation kernel instead; alternatives are
   verifying only on a canonical platform/binary, or accepting per-platform
   authority.
5. **State checksums.** Hash the full simulation state every N ticks and record
   it with the replay; on re-run, compare. This converts "mystery desync weeks
   later" into "divergence first appeared at tick 4 231," which is debuggable.
6. **Deterministic dependencies only.** Third-party physics/pathfinding
   libraries are frequently *not* deterministic (internal RNG, threading,
   platform-specific SIMD paths) — either verify and pin them, or keep them
   out of the kernel and use them only for presentation.

#### Benefits
- Replay, offline recompute, PvP verification, and lockstep/rollback netcode
  all become the *same* trusted mechanism rather than four separate systems.
- Bug reports can ship as a seed + command log that reproduces exactly.
- Determinism tests (§17) catch whole classes of logic bugs (order-dependence,
  uninitialized state) that ordinary unit tests miss.

#### Costs & trade-offs
- Fixed-point/integer math is less ergonomic than floats and needs range/
  precision design; retrofitting it into a float-based kernel is a rewrite.
- Stable iteration and per-concern RNG streams add bookkeeping and forbid some
  convenient data structures inside the kernel.
- Determinism constrains parallelism: multithreading the kernel requires
  deterministic scheduling/reduction, which is significant extra work.

#### When to use
Whenever any of §1's headless use cases, §3's replay, §12's verification, or
§13's lockstep/rollback models is a requirement. Adopt from the first line of
simulation code — this is the canonical "cheap now, prohibitive later" property.

#### When not to use
A purely cosmetic system, or a single-player game with no replay/recompute/
verification surface, doesn't need bit-exactness — approximate reproducibility
(same seed → same *content*) may be enough, at much lower cost.

#### Decision criteria
- Must two independent runs produce *bit-identical* results (verification,
  lockstep, rollback)? Full rules 1–6 apply.
- Must results merely be *plausibly reproducible* (debugging convenience)?
  Rules 1–3 alone give most of the value.
- Will verification re-runs happen on different hardware/OS/builds than the
  original run? Rule 4 forces a decision now: integer/fixed-point kernel, or
  canonical-platform verification.

#### Common mistakes
- One shared global RNG used by both simulation and presentation — a new
  particle effect changes combat outcomes.
- Iterating a hash map "because it's the natural container," producing results
  that differ between runs, platforms, or language versions.
- Assuming float determinism because tests pass on one machine — divergence
  appears only across compilers/architectures/optimization levels.
- Trusting an engine's built-in physics inside the kernel without verifying
  its determinism guarantees (most engine physics is not cross-platform
  deterministic, and often not even run-to-run deterministic).

#### Related patterns
Fixed-tick pipeline (§2) — the timing half of determinism; command pattern
(§3) — the input half; async PvP verification (§12) and lockstep/rollback
(§13) — the consumers; golden-replay testing (§17) — the enforcement.

#### Sources
Glenn Fiedler, "Deterministic Lockstep" and "Floating Point Determinism";
GGPO rollback SDK (determinism as the precondition for rollback netcode);
production deterministic-simulation engines using fixed-point math for
cross-platform consistency.

---

## 5. Entity Modeling: ECS vs Scene-Graph/Node Composition

#### Summary
Two dominant ways to structure game objects: **Entity-Component-System (ECS)** —
data-oriented, entities are IDs with attached component data, systems operate
over component arrays — versus **scene-graph/node composition** — object-
oriented, each game object is a tree of composed nodes/components with
behavior attached directly.

#### Problem it addresses
Game objects need both varied behavior (a character has stats, skills, status
effects, AI, rendering) and, at scale, performance (iterating thousands of
entities' movement/combat logic per tick). Deep inheritance hierarchies
("Unit → Hero → RangedHero → ArcherHero") become rigid and don't compose well;
a flat, undifferentiated update loop over "everything" doesn't scale or stay
organized either.

#### Description / How it works

**ECS:** Entities are lightweight IDs. Components are pure data (a `Health`
component, a `Position` component, a `StatusEffects` component) with no
behavior. Systems are functions that iterate over all entities possessing a
given set of components and operate on them in bulk (a `DamageSystem` iterates
every entity with `Health` + pending damage events). Composition is "what
components does this entity have," not inheritance.

**Scene-graph/node composition:** Each game object is a node (or a small tree
of child nodes/components) in a hierarchical scene structure; behavior is
attached via composed child nodes/components rather than a monolithic
inheritance chain (a `HealthComponent` node, a `StatusEffectComponent` node, an
`AIComponent` node, composed under an `Enemy` scene) — favoring **composition
over inheritance** ([`03` §2.6](03-software-design-principles.md#26-composition-over-inheritance))
within the engine's native object/scene model rather than adopting a separate
ECS data model. Note the terminology trap: engine "component" systems where
components carry behavior (Unity `GameObject`s, engine node trees) are
**entity-component (EC) frameworks**, not ECS — the defining ECS property is
the separation of data (components) from behavior (systems).

**ECS implementation families** (relevant when adopting or evaluating an ECS
library): *archetype/table-based* storage (entities grouped in tables by
component set — fastest bulk queries/iteration; used by most current
large-scale implementations) vs *sparse-set* storage (per-component sparse
arrays — fastest add/remove of components). The trade-off is iteration speed
vs. structural-change speed; both are mainstream in production engines and
games.

#### Benefits

*ECS:* Excellent cache locality and bulk-iteration performance at high entity
counts (thousands+), following data-oriented design (§14); trivially easy to
add a new cross-cutting system without touching entity definitions; naturally
data-oriented, which pairs well with data-driven content (§8) and with
deterministic, order-explicit iteration (§4 rule 2 requires exactly this kind
of explicit, stable iteration order).

*Scene-graph/node composition:* Matches most game engines' native authoring
tools directly (visual scene editors, inspector-driven composition), so
designers can assemble variants without code; simpler mental model for small-
to-moderate entity counts; less architectural machinery to build/maintain.

#### Costs & trade-offs

*ECS:* More upfront architecture (component storage, system scheduling); less
natural fit with an engine's built-in scene editor/inspector workflow unless
the engine has first-class ECS support; can feel like over-engineering below
the entity-count threshold where its performance benefit matters. Writing a
custom ECS is easy to start and hard to make competitive with mature libraries.

*Scene-graph/node composition:* Node-per-entity overhead (tree traversal,
per-node dispatch) scales worse at very high entity counts than tightly-packed
component arrays; behavior can still creep toward inheritance-chain rigidity
if composition discipline lapses.

#### When to use

*ECS:* Large entity counts with performance-critical bulk updates (large-scale
battles, particle-like swarms, colony/city simulations); when the same set of
cross-cutting systems (movement, damage, status) must apply uniformly across
very different entity kinds. Production examples span shooters, city builders,
and large-scale voxel/sandbox games.

*Scene-graph/node composition:* Small-to-moderate entity counts (a party-based
combat game with a handful of player characters and a bounded per-fight enemy
count is solidly in this range); when leveraging the engine's native
editor/authoring tools for composition is valuable; when team size/expertise
favors the engine's default object model over building/learning a separate ECS
layer.

#### When not to use
ECS is not worth its architectural cost for entity counts in the tens (not
thousands) with no bulk-iteration performance problem — the added indirection
buys nothing there. Pure inheritance-chain node hierarchies (the thing
composition is meant to replace) should be avoided regardless of which model
is chosen.

#### Decision criteria
- What is the peak simultaneous entity count the update loop must handle? Low
  (tens) → node composition is simpler and sufficient. High (thousands) → ECS's
  performance profile starts to matter.
- Does the team/engine have strong native scene-authoring tools worth keeping?
  Favors node composition (or a "component" pattern within the engine's own
  object model, capturing ECS's *composition* benefit without a full data-
  oriented rewrite).
- Is bulk, uniform, cross-cutting iteration (damage resolution across every
  active entity, every tick) the dominant cost? Favors ECS's cache-friendly
  layout.
- If adopting ECS: prefer a mature library over a bespoke implementation
  unless the project's requirements are genuinely unusual.

#### Common mistakes
- Building a full custom ECS for a small, bounded entity count "because it's
  the modern pattern" — pure resume-driven complexity (mirrors the "Golden
  Hammer" anti-pattern, [`02` §10](02-architecture-patterns.md#10-architectural-anti-patterns)).
- Letting a node-composition hierarchy quietly regrow into deep inheritance
  ("Enemy → EliteEnemy → EliteFireEnemy → EliteFireBossEnemy") instead of
  composing behavior via sibling/child components.
- In either model, iterating entities in a non-deterministic order (hash-map/
  set-native order) when the simulation must be reproducible — an entity-
  modeling choice does not exempt the system from the ordering discipline in
  §4.

#### Related patterns
Composition over inheritance ([`03` §2.6](03-software-design-principles.md#26-composition-over-inheritance));
data-driven content pipeline (§8); data locality (§14); deterministic update
ordering (§4).

#### Sources
Robert Nystrom, *Game Programming Patterns* — Component; Adam Martin, "Entity
Systems are the future of MMOG development"; Scott Bilas, "A Data-Driven Game
Object System" (GDC 2002); Sander Mertens, *ECS FAQ* (implementation families,
EC-vs-ECS distinction); Blizzard, "Overwatch Gameplay Architecture and Netcode"
(GDC 2017).

---

## 6. State Machines for Game Logic

#### Summary
Model discrete-mode game logic — combat phases, AI behavior, UI flow, a
character's action state — as explicit **finite state machines (FSMs)** or
**hierarchical/behavior-tree variants**, rather than networks of boolean flags.

#### Problem it addresses
"Is the character currently casting, stunned, dead, or channeling?" answered by
a scatter of independent booleans (`is_casting`, `is_stunned`, `is_dead`,
`is_channeling`) admits invalid combinations (stunned *and* casting) and grows
combinatorially unreadable. A state machine makes the valid states and
transitions explicit and enforces that only one state (or a well-defined
hierarchy of states) is active at a time.

#### Description / How it works
```
        cast_started          hit_confirmed
  Idle ───────────────▶ Casting ───────────▶ Idle
    ▲                      │  stun_applied
    │  stun_expires        ▼
    └──────────────── Stunned
```
A state machine defines a fixed set of states and the legal transitions between
them, each triggered by an event/command (tying back to §3). **Hierarchical
state machines** nest states (e.g. "Alive" containing "Idle/Casting/Moving,"
with "Dead" as a sibling top-level state) so common transitions (any
Alive-substate → Dead) are defined once. **Behavior trees** generalize this for
AI decision-making, composing selector/sequence/condition nodes into more
flexible branching logic than a flat FSM handles well (see §7 for the wider
AI-architecture menu).

#### Benefits
- Invalid state combinations become structurally impossible rather than a bug
  to avoid by convention.
- Transitions are an explicit, reviewable list — "what can happen from here"
  is answerable by reading the state's transition table, not by grepping for
  every place a flag is set.
- Pairs naturally with the command pattern (§3): a command's legality can be
  checked against the current state before it's applied.

#### Costs & trade-offs
- More upfront modeling than "just add a boolean" for the first flag or two —
  the payoff shows up as the number of interacting states grows.
- A flat FSM can explode combinatorially for genuinely orthogonal concerns
  (e.g. "casting" × "poisoned" × "in melee range" are independent axes, not
  one state machine) — model orthogonal concerns as separate, composed state
  machines or as data (status-effect list) rather than forcing one FSM to
  represent every combination.

#### When to use
Any entity or system with mutually-exclusive modes and meaningful transition
logic: combat action state, AI behavior, UI screen flow, a scripted automation
rule's own evaluation state if it has one. Behavior trees specifically for AI
decision-making richer than a simple mode switch (enemy AI choosing among
several viable actions based on conditions).

#### When not to use
A single independent boolean flag with no interacting transitions (a simple
on/off toggle with no other state depending on it) doesn't need a state-machine
frame — that would be over-engineering a light switch.

#### Decision criteria
Are there ≥3 mutually exclusive modes, or does one flag's value change what
transitions are legal for another? If yes, model it as a state machine. Is the
decision logic closer to "pick the best of several viable actions given
conditions" (AI) than "which single mode am I in"? Favor a behavior tree or one
of the decision architectures in §7 over a flat FSM.

#### Common mistakes
- Modeling orthogonal, simultaneously-true concerns (multiple simultaneous
  status effects) as if they were mutually-exclusive states of one FSM.
- Letting transition logic leak outside the state machine (code elsewhere
  directly flips what should be a state-machine-owned mode), reintroducing the
  invalid-combination problem the FSM was meant to prevent.
- No default/error transition for an unexpected event in a given state —
  causes silent no-ops or crashes on edge cases instead of a defined fallback.

#### Related patterns
Command pattern (§3) — commands are natural transition triggers; state design
pattern (GoF); AI decision architectures (§7).

#### Sources
Robert Nystrom, *Game Programming Patterns* — State; GoF *Design Patterns* —
State pattern; Damian Isla, "Handling Complexity in the Halo 2 AI" (GDC 2005).

---

## 7. Game AI Decision Architectures

#### Summary
Beyond FSMs and behavior trees (§6), game AI has a small standard menu of
decision architectures — **utility AI**, **planners (GOAP/HTN)**, **influence
maps**, **steering behaviors**, and **search (MCTS)** — each fitting a
different shape of decision problem. Choosing the architecture to match the
problem shape matters more than any single technique.

#### Problem it addresses
As AI requirements grow ("pick the best target considering health, range,
threat, and role," "coordinate a group," "plan a multi-step action sequence"),
a flat FSM or hand-tuned if/else cascade becomes unmaintainable: every new
consideration multiplies branches, and tuning one behavior breaks another.

#### Description / How it works
- **Behavior trees (BT):** hierarchical composition of selector/sequence/
  condition/action nodes; the industry default for reactive character behavior.
  Strong for authorable, designer-readable reactive logic; weak at weighing
  many continuous factors.
- **Utility AI:** score every candidate action from weighted *considerations*
  (normalized curves over inputs like distance, health, cooldowns) and pick the
  best (or weighted-random among top scorers). Strong when many factors trade
  off continuously; scoring curves are data (§8) and can be tuned/tested in
  bulk. Weak at multi-step lookahead.
- **Planners (GOAP / HTN):** the AI is given goals and a library of actions
  with preconditions/effects (GOAP) or hierarchically-decomposed tasks (HTN);
  a planner searches for an action sequence at runtime. Strong for emergent
  multi-step behavior with less authoring of explicit transitions; costs
  runtime search and is harder to predict/debug.
- **Influence maps:** spatial grids/fields aggregating threat, control, or
  desirability; queried by any of the above for positional decisions ("where
  is safe," "where to flank"). A shared spatial-reasoning substrate, not a
  decision-maker by itself.
- **Steering behaviors:** local movement decisions (seek/flee/avoid/flock,
  context steering, flow fields for crowds) — the movement layer under
  whatever decision layer sits above.
- **Search (MCTS/minimax):** for discrete, rule-complete decision spaces
  (turn-based tactics, card games) where simulating candidate futures is
  feasible — often using the same deterministic simulation core (§1, §4) as a
  lookahead engine.

These compose: a typical production stack is BT or utility for action
selection, a planner for long-horizon goals where needed, influence maps for
spatial queries, and steering underneath for movement.

#### Benefits
- Matching architecture to problem shape keeps each behavior small, testable,
  and independently tunable.
- Utility considerations and BT structures can live as data (§8), enabling
  designer iteration and bulk balance testing without code changes.
- A deterministic sim core (§4) lets search-based AI and automated balance
  agents reuse the real rules rather than a parallel approximation.

#### Costs & trade-offs
- Each additional architecture is a competence the team must maintain; a
  mixed stack is powerful but harder to reason about than one mediocre-but-
  uniform approach.
- Planners and search trade authoring effort for runtime cost and debugging
  opacity ("why did it choose that?" needs dedicated tooling/logging).
- Utility AI's flexibility is also its failure mode: badly-normalized curves
  produce confident nonsense that is hard to spot in review.

#### When to use
- ≤ a handful of discrete modes → FSM (§6).
- Reactive, authorable character behavior → BT.
- Many continuously-varying considerations → utility AI.
- Multi-step goal pursuit with a rich action vocabulary → GOAP/HTN.
- Positional/territorial reasoning → influence maps feeding any of the above.
- Rule-complete turn-based decisions with a fast sim → search/MCTS.

#### When not to use
Don't introduce a planner or search where a BT or utility scorer meets the
need — runtime search is the most expensive and least predictable option and
should be pulled in by problem shape, not novelty.

#### Decision criteria
Ask "what does the AI need to weigh?" — discrete modes (FSM), reactive
priorities (BT), continuous trade-offs (utility), action sequencing (planner),
space (influence maps), futures (search). If the answer is "several," compose;
the layers have well-understood seams (decision → movement → animation).

#### Common mistakes
- Scoring functions mixing unnormalized units (raw distance vs. a 0–1 health
  fraction), silently dominating the decision.
- Planner action libraries with under-specified preconditions, producing
  legal-but-absurd plans.
- AI reading presentation-layer state (animation progress, screen positions)
  for decisions — the §1 boundary applies to AI as much as to gameplay logic.
- No decision logging/visualization — any architecture beyond an FSM is
  effectively undebuggable without tooling that answers "why."

#### Related patterns
State machines (§6); data-driven content (§8) for tunable AI parameters;
determinism (§4) for search-based AI and replay-stable behavior; automated
balance agents (§17).

#### Sources
*Game AI Pro* series (free online): "Behavior Selection Algorithms: An
Overview"; "An Introduction to Utility Theory" (Graham); "The Behavior Tree
Starter Kit" (Champandard & Dunstan); "Exploring HTN Planners through Example"
(Humphreys); "Modular Tactical Influence Maps" (Mark); "Context Steering"
(Fray); "Crowd Pathfinding and Steering Using Flow Field Tiles" (Emerson);
"Monte Carlo Tree Search and Related Algorithms for Games" (Sturtevant).
Jeff Orkin's GOAP work (F.E.A.R.); Damian Isla, "Handling Complexity in the
Halo 2 AI" (GDC 2005).

---

## 8. Data-Driven Content Pipeline

#### Summary
Author game content — classes, skills, items, affixes, enemies, level/difficulty
modifiers — as **external data**, loaded and interpreted by generic code,
rather than encoding each piece of content as bespoke source code.

#### Problem it addresses
A game whose content scales primarily through variety and volume (many item
affixes, many skills, procedurally generated character traits, endlessly-scaling
difficulty modifiers) cannot sustainably require a code change and a rebuild for
every new piece of content — content velocity and, where moddability is a goal,
third-party content both depend on content living outside compiled code.

#### Description / How it works
```
   Content authors ──▶ Data files (schema-validated) ──▶ Generic runtime
   (designers, tools)   (classes, skills, items,           code interprets
                         affixes, enemies, modifiers)       the data
```
Define a small number of generic, reusable **data schemas** (a skill has an ID,
a targeting shape, an effect list, numeric parameters; an affix has a stat,
a value range, a rarity weight) and author every specific instance as data
conforming to those schemas. Generic runtime code (a "skill executor," an
"affix applier") interprets the data rather than each skill/affix being its own
hand-written code path. New content is a new data file, not a new code path,
as long as it fits an existing schema; genuinely novel *mechanics* still
require new schema + new interpreter code, but that's a smaller, rarer surface
than "every item."

#### Benefits
- Content volume scales without proportional code growth — critical for a
  design with deep, wide itemization and procedural generation.
- Enables tooling (in-editor content browsers, balance-simulation harnesses
  that iterate every item) that operates generically over "all data of schema
  X" instead of needing per-item special cases.
- A natural foundation for moddability: exposing the same data format to
  players is a much smaller step than exposing source code (with the security
  caveat in Common mistakes).
- Hot-reloading data (seeing a balance-number change without a full rebuild)
  is far more tractable than hot-reloading code.

#### Costs & trade-offs
- Requires schema design discipline upfront — an under-designed schema forces
  either awkward workarounds (cramming a mechanic that doesn't fit the schema)
  or frequent schema-breaking changes.
- Generic interpreter code is initially more work than a bespoke hand-written
  case for the first few pieces of content; the payoff compounds as content
  volume grows.
- Data validation becomes its own responsibility — malformed or
  out-of-range data must fail loudly at load time, not produce silent
  wrong-behavior at runtime.

#### When to use
Any content category with many instances following a common shape (items,
skills, affixes, enemies, procedurally-generated content, dungeon/difficulty
modifiers) — which describes essentially every content axis in a deep-
itemization, procedural-generation-heavy design.

#### When not to use
A one-off, singular piece of bespoke logic (a unique final-boss mechanic that
will never be reused as a pattern) may reasonably stay as hand-written code
rather than forcing a one-instance schema.

#### Decision criteria
Will this content category have more than a handful of instances, or grow via
procedural generation? If yes, model it as data against a schema. Is the
mechanic genuinely one-of-a-kind with no reuse in sight? Hand-written code is
simpler and honest about that.

#### Common mistakes
- Loading untrusted external data (community mods, imported content) as if it
  were trusted, including any data shape that can reference or execute code —
  a moddability feature is a security boundary; restrict external data to
  plain, schema-validated, non-executable formats (many engines' native
  serialized-object formats *can* embed scripts and are therefore unsafe for
  untrusted content).
- No schema versioning — a content-format change breaks every existing data
  file with no migration path, and breaks save compatibility for players whose
  save references now-invalid content IDs (§11).
- Business logic creeping into "data" as ad hoc scripting/expression strings
  evaluated unsafely, effectively becoming an unaudited second scripting
  language.

#### Related patterns
Configuration over code (general software design); ECS's component-as-data
model (§5) pairs naturally with data-driven content; procedural generation
(§9) consumes these schemas; save versioning (§11) depends on stable content
IDs.

#### Sources
General data-driven-design literature in game development; Robert Nystrom,
*Game Programming Patterns* — Type Object; Scott Bilas, "A Data-Driven Game
Object System" (GDC 2002).

---

## 9. Procedural Generation Architecture

#### Summary
Structure procedural content generation as **pure functions of (seed,
parameters)** organized in an explicit **seed hierarchy**, with generated
output validated against the same schemas as authored content (§8) — so
generation is reproducible, testable, and mixable with hand-authored content.

#### Problem it addresses
Ad hoc generation — code sprinkling `random()` calls as it builds a level,
item, or character — produces content that can never be reproduced (bug
reports reference content nobody can see again), can't be regression-tested,
and silently changes whenever any code path adds or removes a random draw.

#### Description / How it works
```
world_seed
  ├── dungeon_stream(seed_d)  ──▶ layout, rooms, connections
  ├── loot_stream(seed_l)     ──▶ item rolls, affixes
  └── unit_stream(seed_u)     ──▶ recruits, traits, names
```
- **Seed hierarchy:** a root seed deterministically derives per-subsystem
  streams (§4 rule 1). Generating one more item never shifts dungeon layout;
  each stream's consumption is isolated.
- **Generation as a pure function:** `generate(seed, params) → content`, with
  no hidden inputs. The same (seed, params, generator version) always yields
  the same content, so a save or replay needs to store only those three
  things, not the generated output (though caching output is a valid
  space/time trade).
- **Constraints and validation:** generated output passes the same
  schema/invariant validation as authored data (§8) plus generation-specific
  constraints (reachability of a level, budget bounds on an item roll).
  Reject-and-reroll or repair strategies must themselves be deterministic.
- **Authored/generated hybrid:** authored templates or grammar rules
  (themselves data, §8) parameterize generation, keeping designer control over
  the possibility space rather than fully-emergent output.

#### Benefits
- Reproducibility: a reported "broken level" is a seed, not a screenshot.
- Testability: golden-seed tests (§17) lock down known-good generation;
  statistical tests over thousands of seeds catch distribution regressions
  (drop rates, difficulty curves).
- Save/replay compactness: store seeds and versions, not content blobs.

#### Costs & trade-offs
- Versioning burden: any generator change silently alters what a stored seed
  produces — saves/replays must record the generator version, and old versions
  must either be kept runnable or their output migrated/cached (§11).
- Pure-function discipline forbids convenient shortcuts (querying live game
  state mid-generation) — inputs must be passed explicitly.
- Constraint systems (reachability, balance budgets) are real engineering,
  and rejection sampling can hide pathological slow cases behind rare seeds.

#### When to use
Any procedurally generated content that players keep (items, characters,
worlds), that appears in saves/replays, or whose distribution needs balancing
— i.e., almost all procedural content beyond momentary cosmetic variation.

#### When not to use
Pure presentation-layer variation (particle jitter, ambient scatter) needs no
seed hierarchy or versioning — just keep it out of the simulation's RNG
streams (§4).

#### Decision criteria
Will anyone ever need this exact output again (save, replay, bug report,
test)? Seeded pure function. Does the output's *distribution* matter to
balance? Add statistical tests. Neither? It's cosmetic; exempt it.

#### Common mistakes
- One global RNG shared across generation subsystems — adding an affix roll
  changes next week's dungeon layouts.
- Storing generated content without the (seed, version) that produced it,
  making "regenerate vs. load" ambiguous after an update.
- Generation reading mutable game state as a hidden input, so the same seed
  produces different content depending on when it runs.
- Validating authored content but trusting generated content — generators
  have bugs too, and an invalid generated item corrupts saves just as surely.

#### Related patterns
Determinism requirements (§4) — generation is a major RNG consumer; data-driven
content (§8) — schemas and templates; save versioning (§11) — generator-version
recording; testing (§17) — golden seeds and statistical tests.

#### Sources
Gillian Smith, "Procedural Content Generation: An Overview" (*Game AI Pro 2*);
Tarn Adams, "Simulation Principles from *Dwarf Fortress*" (*Game AI Pro 2*);
Steve Rabin et al., "Advanced Randomness Techniques for Game AI" (*Game AI
Pro*).

---

## 10. Communication: Events and State Ownership in Games

#### Summary
Prefer **event/signal-driven** communication between decoupled game systems
(a `UnitDefeated` event consumed by loot, XP, and quest systems independently)
over direct cross-system method calls, while keeping each piece of state owned
by exactly one system.

#### Problem it addresses
A combat system that directly calls into the loot system, the XP system, and
the quest-tracking system on every kill couples all four together and makes
adding a fifth consumer (an achievement system) require editing the combat
system again. This is the same coupling problem [`02` §5.3](02-architecture-patterns.md#53-event-driven-architecture-pubsub--event-streaming)
addresses for distributed systems, applied within a single game process.

#### Description / How it works
The simulation core (§1) emits domain events (`UnitDefeated`, `ItemDropped`,
`SkillCast`) from its event log. Independent systems (loot, XP, quest
tracking, achievements, presentation) subscribe to the events they care about
without the emitting system knowing or caring who's listening. Each piece of
state (a unit's HP, an item's ownership, a quest's progress) has exactly one
owning system that mutates it; other systems read via query or react via
event, never mutate another system's owned state directly.

#### Benefits
- New consumers of an existing event (an achievement system reacting to
  `UnitDefeated`) require no change to the emitting system.
- Matches the event log already required for simulation/presentation
  separation (§1) — the same event vocabulary serves both purposes.
- Clear state ownership prevents the "who else might be mutating this" class
  of bug.

#### Costs & trade-offs
- Harder to trace a full causal chain by reading one file — "what happens
  when a unit dies" is now spread across every subscriber, discoverable only
  by finding all listeners of the event.
- Event-carried data must include everything a reasonable subscriber will
  need (event-carried state transfer) or subscribers end up querying back into
  the emitting system anyway, partially defeating the decoupling.

#### When to use
Any cross-system reaction to a game occurrence with more than one interested
consumer, or where the set of consumers is expected to grow (achievements,
telemetry, and quest systems are classic "more consumers arrive later" cases).

#### When not to use
A tight, stable, two-party interaction with no other consumer in sight (e.g. a
UI widget reading its own bound value) doesn't need indirection through an
event bus — direct binding/observation is simpler.

#### Decision criteria
Will more than one system plausibly care about this occurrence, now or later?
Favor events. Is this a private, stable, two-party relationship? Direct
reference/binding is fine.

#### Common mistakes
- Vague, generic events (`StateChanged`) instead of specific, meaningful ones
  (`UnitDefeated`) — mirrors the same anti-pattern in
  [`02` §5.3](02-architecture-patterns.md#53-event-driven-architecture-pubsub--event-streaming).
- Two systems both believing they own the same state and both mutating it.
- Using events for something that actually needs a synchronous, ordered
  answer within the same tick (see §4's ordering requirements) — an
  asynchronous event fan-out is the wrong tool when strict resolution order
  matters; use the command pattern's explicit ordering (§3) for those cases
  instead.

#### Related patterns
Event-Driven Architecture ([`02` §5.3](02-architecture-patterns.md#53-event-driven-architecture-pubsub--event-streaming));
simulation/presentation separation (§1); command pattern (§3).

#### Sources
Same lineage as [`02` §5.3](02-architecture-patterns.md#53-event-driven-architecture-pubsub--event-streaming);
Robert Nystrom, *Game Programming Patterns* — Observer, Event Queue.

---

## 11. Save-Data Persistence & Versioning

#### Summary
Treat save data as a **versioned, migratable data contract** — an explicit
schema with a version number, a migration path from every shipped version, and
corruption-resilient writes — not as a serialized dump of live runtime objects.

#### Problem it addresses
Saves outlive every version of the code that wrote them. A game that
serializes its live object graph directly ties save compatibility to internal
refactors (rename a field, break every player's save), and a game without
migrations strands players on every content update. Save corruption (crash or
power loss mid-write) with no recovery path destroys the most valuable thing a
player owns.

#### Description / How it works
- **Explicit save schema:** save data is a designed, documented format
  (versioned like the content schemas in §8, and referencing content by
  stable ID, never by array index or memory reference). The mapping between
  runtime state and save format is deliberate code, so internal refactors
  don't silently change the format.
- **Version + migration chain:** every save records its schema version;
  loading applies stepwise migrations (v3→v4→v5) so any shipped version
  remains loadable. Migrations are tested against a retained corpus of real
  saves from each shipped version (§17).
- **Atomic, corruption-resilient writes:** write to a temporary file, flush,
  then rename over the previous save; optionally keep N rotating backups.
  Validate on load (checksums, schema validation) and fall back to the newest
  valid backup rather than crashing.
- **Scope separation:** settings, meta-progression, and world/session state
  have different lifecycles and risk profiles — separate files/records so a
  corrupt session save doesn't take account-level progression with it.
- **Untrusted input:** a save file is externally-modifiable input. Loading
  must be robust against malformed data, and save formats must not be able to
  reference or execute code (same rule as §8's mod data).
- **Procedural content:** store (seed, generator version, params) rather than
  generated blobs where practical (§9), plus any player-mutated deltas.

#### Benefits
- Content updates and refactors stop being save-breaking events — player
  trust and review scores depend on this more than almost any other
  reliability property.
- Corruption recovery turns "lost 80 hours" into "lost 5 minutes."
- A designed save format is also the foundation for cloud sync, cross-device
  play, and server-side verification of client-reported progression.

#### Costs & trade-offs
- Explicit mapping + migrations is permanent ongoing work — every schema
  change now has a migration cost, which is precisely the point but must be
  budgeted.
- A retained save corpus and migration tests add CI weight.
- Storing seeds instead of content blobs binds you to keeping old generator
  versions runnable (or migrating their output) — see §9's versioning note.

#### When to use
Every game with persistence beyond a single session. The discipline scales
down (a small game may need one version field, one migration, one backup) but
never to zero.

#### When not to use
Only truly ephemeral state (a score attack with a leaderboard entry and no
saves) escapes this; even settings files benefit from the atomic-write rule.

#### Decision criteria
- Will players accumulate state they'd be upset to lose? Full discipline:
  versioning + migrations + atomic writes + backups.
- Is save data ever sent to a server or shared between players? Add strict
  validation and treat it as untrusted input on receipt.
- Does content evolve after launch? Stable content IDs (§8) and a policy for
  saves referencing removed content (graceful degradation, substitution, or
  compensation) are required, not optional.

#### Common mistakes
- Serializing engine objects / live object graphs directly — couples the save
  format to code layout, engine version, and (in many engines) opens the
  can-embed-code security hole.
- No version field "because we'll add it when we need it" — the first save
  without a version is unmigratable by definition.
- In-place overwrite of the only save file, so a crash mid-write is total
  loss.
- Testing migrations only against synthetic saves, not real ones from shipped
  builds — real saves contain the states you forgot were reachable.

#### Related patterns
Data-driven content (§8) — stable content IDs; procedural generation (§9) —
seed/version storage; command pattern (§3) — some designs persist the command
log itself (event-sourced saves, [`02` §6.4](02-architecture-patterns.md#64-event-sourcing));
database change management parallels in [`07`](07-security-reliability-operations.md).

#### Sources
Jason Gregory, *Game Engine Architecture* (serialization and save-game
systems); widely-documented industry practice (atomic save writes, save
versioning/migration); same schema-evolution reasoning as
[`02` §6.4](02-architecture-patterns.md#64-event-sourcing).

---

## 12. Netcode Patterns for Asynchronous / Replay-Based Multiplayer

#### Summary
For multiplayer built on **asynchronous, snapshot/replay-based** interaction
(one player's party fights a snapshot of another's, rather than both playing
live simultaneously) — as opposed to real-time lockstep or client-server
live-action netcode — the architecture is closer to "replay the deterministic
simulation with a captured opponent input set" than to traditional real-time
networking.

#### Problem it addresses
Live real-time multiplayer (lockstep, rollback, authoritative-server tick
sync — see §13) solves a much harder problem than async competition needs, at
much higher cost: async PvP only requires that a fight between a live player's
current build and a *stored snapshot* of another player's build produces a
fair, reproducible, tamper-resistant result — no live connection between the
two players is ever required.

#### Description / How it works
```
Player A's party (live)          Player B's party (snapshot, stored server-side)
         │                                    │
         └──────────────┬─────────────────────┘
                         ▼
         Deterministic simulation core (§1)
         runs once, using both parties' data
         as input, with a fixed/shared seed
                         │
                         ▼
              Result + full event log
     (verifiable by re-running the same simulation
      with the same inputs, server-side or client-side)
```
Each player's party/build (characters, gear, stats, automation rules) is
serialized as a snapshot. An async PvP fight is a single deterministic
simulation run (§1–§4) over both snapshots — computed once, its result is
final and doesn't require either player to be online simultaneously.
**Fairness and anti-cheat both reduce to determinism**: because the simulation
is deterministic and both parties' data are captured/stored, any party
(server, opponent's client, a third-party auditor) can independently
re-run the exact same simulation and must get the exact same result — a
mismatch is proof of tampering or a non-determinism bug, not a legitimate
outcome to accept.

#### Benefits
- Avoids the entire class of real-time-networking problems (latency
  compensation, rollback, live disconnect handling) that live multiplayer
  requires — the "netcode" here is really "serialize a snapshot and replay a
  deterministic sim," reusing the exact same core built for offline-progress
  calculation (§1, §2).
- Server-side (or independently-computed) re-simulation is a direct anti-cheat
  mechanism: a client-reported result that doesn't match an independent replay
  is rejected outright, with no heuristic cheat-detection needed.
- Scales trivially compared to live matchmaking — no session/connection
  management, no server-side game-hosting per match.

#### Costs & trade-offs
- Only fits interaction models that genuinely don't need live simultaneous
  play — it cannot become real-time PvP later without a substantially
  different netcode layer (§13).
- Snapshot staleness is a design question: if Player B improves their party
  after being snapshotted, defenders may be fighting an out-of-date version of
  themselves — needs an explicit policy (refresh cadence, a defense loadout
  the player actively curates, etc.), not just an implementation detail.
- Full trust in the determinism requirement (§4) — any non-determinism bug
  becomes a fairness/anti-cheat hole, not just a visual glitch. In particular,
  if verification re-runs happen on different hardware than the original run
  (server x86 vs. client mobile ARM), floating-point divergence (§4 rule 4)
  will produce false tampering verdicts — an integer/fixed-point kernel or a
  single canonical verification platform is required.

#### When to use
Async PvP (arena/defense-loadout formats), leaderboard-driven competition,
and any "my build vs. a stored snapshot of your build" interaction model —
a common shape across mobile and idle/RPG genres.

#### When not to use
Live, simultaneous, latency-sensitive multiplayer (real-time co-op, live PvP
where both players act and react to each other in the moment) needs the
lockstep/rollback/authoritative-server models in §13 instead — a fundamentally
different, more expensive architecture this pattern does not cover.

#### Decision criteria
Do both participants need to be online and reacting to each other live? If
yes, this pattern doesn't apply — see §13. Is the interaction "my current
build vs. a captured snapshot," resolved as a single deterministic
computation? This pattern is the direct, low-cost fit.

#### Common mistakes
- Treating the client-computed result as authoritative without an independent
  re-simulation to verify it — reopens exactly the cheating vector determinism
  was meant to close.
- No defined snapshot-refresh/staleness policy, leading to confusing or unfair
  "defense" outcomes players can't reason about.
- Allowing any non-deterministic element (wall-clock-seeded RNG, unordered
  iteration, floating-point order-dependence) into the combat kernel — this
  pattern has zero tolerance for the anti-patterns catalogued in §4.

#### Related patterns
Command pattern (§3) — the replay mechanism; simulation/presentation
separation (§1); determinism requirements (§4); real-time netcode models
(§13); the general distributed-systems idempotency/verification reasoning in
[`02` §7.7](02-architecture-patterns.md#77-idempotency).

#### Sources
Glenn Fiedler, "Deterministic Lockstep" (the general determinism-for-
multiplayer reasoning, applied here to the async/snapshot case specifically);
general async-PvP/"defense loadout" pattern as used across mobile idle/RPG
games.

---

## 13. Real-Time Netcode Models

#### Summary
Live multiplayer has four standard architectural families — **deterministic
lockstep**, **rollback**, **authoritative server with client-side prediction**,
and **snapshot interpolation** — differing in what is transmitted (inputs vs
state), where authority lives, and how latency is hidden. Choosing among them
is a bandwidth / latency-feel / cheat-resistance / determinism-cost trade.

#### Problem it addresses
Network latency is physics: inputs take tens to hundreds of milliseconds to
travel, but players expect instant response and a consistent shared world.
Every real-time netcode model is a different answer to "who simulates, what is
sent, and how is the delay hidden" — picking the wrong model for the genre
produces either unplayable input lag, unaffordable bandwidth, or trivially
exploitable cheating.

#### Description / How it works
- **Deterministic lockstep** — send only each player's *inputs*; every peer
  runs the same deterministic simulation (§4). Bandwidth is proportional to
  input size, not world size (a million-object sim networks as cheaply as
  one). The simulation cannot advance past a tick until all inputs for it have
  arrived, so latency/jitter appear as input delay or stalls (mitigated by a
  playout-delay buffer and by sending redundant un-acked inputs over UDP
  rather than waiting on TCP retransmission). Classic fit: RTS, large unit
  counts, same-build peers.
- **Rollback** — lockstep's responsiveness fix: *predict* remote inputs
  (usually "same as last tick"), simulate immediately, and on receiving the
  real input, roll back to the last confirmed state and re-simulate forward.
  Local input feels zero-latency; mispredictions appear as small corrections.
  Requires a deterministic sim (§4) *plus* fast state save/load and the CPU
  headroom to re-simulate several ticks per frame. Industry standard for
  fighting games; increasingly the expected baseline for peer-to-peer action.
- **Authoritative server + client-side prediction** — clients send inputs;
  the server runs the authoritative simulation and broadcasts state. Clients
  *predict* their own movement locally and **reconcile** when the server's
  authoritative result arrives (replaying pending inputs on top of it); other
  players are displayed via **entity interpolation** between received states
  (slightly in the past); optional **lag compensation** has the server rewind
  its world to what a shooting client actually saw when validating hits.
  The standard for FPS/action games and anything cheat-sensitive: the server
  is the single source of truth and clients are untrusted input devices.
- **Snapshot interpolation** — the server (or session host) sends compressed
  world-state snapshots at a fixed rate; clients buffer and interpolate
  between them without simulating game logic locally. Simple and robust
  (no client sim, no determinism requirement), at the cost of bandwidth
  scaling with world size and all interaction feeling one buffer behind —
  often combined with prediction for the local player only.

Security holds across all models: **never trust the client.** Validate every
client-supplied input server-side (or peer-side), rate-limit, and treat
client-reported state (position, cooldowns, results) as a cheat vector — the
same reasoning as §12's verification-by-re-simulation.

#### Benefits
- Each model is well-documented, battle-tested, and has known genre fits —
  this is one of the few game-architecture areas with genuine industry
  consensus.
- Input-based models (lockstep/rollback) give huge-world bandwidth savings
  and replay/verification for free (the input log *is* the replay, §3).
- Server-authoritative models give cheat resistance and late-join/reconnect
  simplicity (state lives in one place).

#### Costs & trade-offs
- Lockstep/rollback require full determinism (§4) including its cross-platform
  float problem, and make late-join/reconnect hard (must transfer and verify
  full state).
- Rollback's re-simulation budget constrains simulation cost per tick; visual
  corrections need concealment work (animation smoothing).
- Server-authoritative models cost server capacity, add prediction/
  reconciliation complexity per gameplay mechanic, and lag compensation
  creates "shot behind cover" controversies — a designed trade, not a bug.
- Snapshot models spend bandwidth proportional to visible world size and need
  aggressive compression/delta encoding at scale.

#### When to use
- Many entities, few players, same build on all peers → lockstep.
- Few entities, twitch-sensitive, peer-to-peer → rollback.
- Cheat-sensitive, many players, mixed platforms → authoritative server +
  prediction.
- Physics-heavy or logic-light worlds, or thin clients → snapshot
  interpolation (+ local prediction as needed).

#### When not to use
Don't build live netcode at all if the design is satisfied by asynchronous
snapshot competition (§12) — it is an order of magnitude cheaper. Don't use
client-authority "temporarily" in a competitive game; it never gets removed
under schedule pressure.

#### Decision criteria
Four questions pick the model: (1) How much state would state-transfer cost vs
input-transfer (world size)? (2) How latency-sensitive is the core verb
(twitch vs strategic)? (3) How cheat-sensitive is the game (competitive/
economy vs co-op among friends)? (4) Can the team afford determinism (§4)
across all shipping platforms? The answers usually leave exactly one viable
family.

#### Common mistakes
- Retrofitting multiplayer onto a simulation that reads render state, uses
  ambient RNG, or has no fixed tick — the §1/§2/§4 disciplines are the actual
  prerequisite for every model except pure snapshot interpolation.
- Sending reliable-ordered everything (TCP or reliable channels) for
  latency-sensitive data — one lost packet stalls everything behind it;
  unreliable delivery with redundancy or sequencing is the standard fix.
- Testing only on LAN — every model's failure modes (buffer stalls, rollback
  depth, reconciliation snaps) appear only under real latency, jitter, and
  loss; test with network-condition simulation from day one.
- Letting clients report outcomes ("I hit him," "I have 100 gold") instead of
  inputs/intents that the authority validates.

#### Related patterns
Determinism requirements (§4) — precondition for lockstep/rollback; command
pattern (§3) — inputs-as-data; async alternative (§12); event-driven state
distribution parallels in [`02` §5.3](02-architecture-patterns.md#53-event-driven-architecture-pubsub--event-streaming).

#### Sources
Gabriel Gambetta, *Fast-Paced Multiplayer* series (authoritative server,
prediction/reconciliation, entity interpolation, lag compensation); Glenn
Fiedler, "Deterministic Lockstep," "Snapshot Interpolation," and the
networked-physics series; GGPO rollback SDK (Tony Cannon); Blizzard,
"Overwatch Gameplay Architecture and Netcode" (GDC 2017).

---

## 14. Performance Patterns (Pooling, Spatial Partitioning, Data Locality)

#### Summary
Four standard patterns cover most game-side CPU performance work: **object
pooling** (reuse instead of allocate), **spatial partitioning** (query by
location instead of scanning everything), **data locality** (lay data out for
the cache, the core idea of data-oriented design), and **dirty flags** (compute
only what changed) — plus **time-slicing/LOD** for work that can be amortized.

#### Problem it addresses
Per-tick simulation work scales with entity count, and frame budgets (§15) are
hard deadlines. The dominant, recurring causes of blown budgets are: allocation
/garbage-collection spikes from per-frame object churn; O(n²) proximity scans
("check every entity against every other"); cache-hostile memory layouts
(pointer-chasing scattered objects); and recomputing expensive derived values
that didn't change.

#### Description / How it works
- **Object pool:** pre-allocate a fixed set of reusable instances
  (projectiles, particles, damage numbers, network packets); "creating" one
  takes it from the pool, "destroying" returns it. Eliminates both allocation
  cost and GC/fragmentation pressure in managed and native runtimes alike.
  Requires explicit reset-on-reuse and a policy for pool exhaustion (grow,
  drop-oldest, or fail loudly).
- **Spatial partition:** a uniform grid, quadtree/octree, or BVH mapping
  positions to entities, so "what's within radius r" is a local lookup, not a
  full scan. A uniform grid is usually the right first choice (simple,
  rebuild-per-tick is often cheaper than incremental maintenance); trees earn
  their complexity with strongly non-uniform entity distributions.
- **Data locality / data-oriented design:** iterate contiguous arrays of
  exactly the data the loop needs (SoA — struct-of-arrays) rather than
  pointer-chasing heterogeneous objects (AoS with cold fields). Split hot data
  (touched every tick) from cold data (touched rarely); this is the mechanism
  behind ECS's bulk-iteration speed (§5) but applies without ECS too.
- **Dirty flag:** cache derived values (world transforms, aggregated stats,
  pathfinding results) and recompute only when a dependency actually changed.
  The cost is the discipline of invalidating correctly — a missed invalidation
  is a stale-value bug.
- **Time-slicing / logic LOD:** not everything needs every tick — update
  distant/off-screen AI at a lower rate, spread expensive scans across N
  ticks, budget pathfinding requests per frame. In a deterministic kernel
  (§4), slicing schedules must themselves be deterministic (tick-count based,
  not wall-time based).

#### Benefits
- These four patterns account for the large majority of "game got slow at
  scale" fixes and are cheap to apply where the design anticipated them.
- Pooling converts unpredictable GC/allocator spikes into predictable, flat
  cost — often the difference between passing and failing frame pacing (§15).
- Spatial partitioning turns the single most common O(n²) hotspot into O(n).

#### Costs & trade-offs
- Every pattern adds bookkeeping and a new bug class: stale pooled state,
  entities in the wrong partition cell after moving, missed dirty-flag
  invalidation, LOD'd AI visibly "waking up."
- Data-oriented layouts trade ergonomics for speed — SoA code is more awkward
  to write and refactor than object-oriented AoS.
- Applied before profiling shows the need (§15), they are premature
  complexity; the entity counts of many games never reach the threshold.

#### When to use
When profiling (§15) attributes frame cost to allocation churn, proximity
scans, cache misses on bulk iteration, or repeated derived-value computation —
or at design time for systems *known* to face high counts (bullets, particles,
large battles, crowd sims).

#### When not to use
At prototype scale, or for systems with bounded small counts — a fight with
eight units needs none of this, and the bookkeeping would only add bug
surface.

#### Decision criteria
Measure first (§15). Then: allocation spikes → pool; proximity queries →
spatial partition; bulk-iteration cache misses → locality/SoA (possibly full
ECS, §5); repeated derived computation → dirty flag; "everything a little too
expensive" → time-slicing and logic LOD before micro-optimization.

#### Common mistakes
- Pooled objects leaking state between uses (the previous owner's target,
  a still-subscribed event handler) — reset must be total and tested.
- Rebuilding a complex tree structure every tick when a dumb uniform grid
  would be both simpler and faster.
- Optimizing the object model around cache behavior before any measurement,
  in a game whose real bottleneck is the GPU or a single pathological
  algorithm.
- Time-slicing gameplay-visible logic on wall-clock time inside a
  deterministic kernel, breaking §4.

#### Related patterns
Entity modeling (§5) — ECS is data locality institutionalized; frame budget &
profiling (§15) — the evidence for applying any of these; determinism (§4) —
constraints on slicing/scheduling.

#### Sources
Robert Nystrom, *Game Programming Patterns* — Object Pool, Spatial Partition,
Data Locality, Dirty Flag; Richard Fabian, *Data-Oriented Design*; Mike Acton,
"Data-Oriented Design and C++" (CppCon 2014); DICE, "Culling the Battlefield"
(GDC 2011).

---

## 15. Frame-Budget & Profiling Discipline

#### Summary
Treat the frame as a **hard real-time budget** (16.6 ms at 60 Hz) allocated
explicitly across subsystems, enforced by continuous measurement on target
hardware — not as an average FPS number checked occasionally on a developer
machine.

#### Problem it addresses
Frame cost grows by accretion: each feature adds "only a millisecond," nobody
owns the total, and the game discovers it misses 60 Hz — or thermally throttles
on mobile — months later, when the causes are hundreds of small, entangled
costs. Average FPS hides the real player experience: intermittent spikes
(hitches) are more noticeable than a uniformly lower frame rate.

#### Description / How it works
- **Explicit budget allocation:** divide the frame across subsystems
  (simulation tick, AI, physics, animation, rendering submission, UI, audio)
  with named owners — the same structure as any capacity budget. A subsystem
  over budget is a bug with an owner, not ambient slowness.
- **Measure the distribution, not the mean:** track frame-time percentiles
  and worst-frame spikes (e.g. "1% low" frame times); a 200 ms hitch every
  ten seconds ruins a 60 FPS average. Frame *pacing* (consistent delivery)
  matters as much as frame *rate* — a stable 30 is smoother than an erratic
  45.
- **Profile release builds on target hardware:** debug builds and developer
  machines misattribute costs. Mobile adds a second, sustained dimension:
  a device that holds 60 Hz for two minutes may thermally throttle to a
  fraction of that after twenty — soak-test, don't spot-check.
- **Instrument continuously:** lightweight per-subsystem timers shipping in
  development builds (and ideally telemetry from real devices) catch budget
  regressions at commit time; deep profilers (sampling/tracing) are for
  diagnosis, not detection.
- **Degradation levers:** decide *in advance* what sheds load when over
  budget — render resolution/quality scaling, logic LOD and time-slicing
  (§14), effect density caps — rather than discovering at ship time that
  nothing is detachable.

#### Benefits
- Converts "the game feels bad sometimes" into an attributable, regressable
  metric per subsystem.
- Budget ownership makes performance a design-time constraint on features
  ("this system gets 2 ms") instead of a post-hoc crisis.
- Percentile/pacing focus targets what players actually perceive.

#### Costs & trade-offs
- Instrumentation and device-lab/soak testing are ongoing infrastructure
  costs, easy to deprioritize because their absence hurts only later.
- Hard budgets create real design pressure — features get cut or simplified
  against them; that is the mechanism working, but it needs organizational
  buy-in.
- Over-instrumentation itself costs frame time; measurement must be cheap and
  toggleable.

#### When to use
Any game with a real-time rendering loop, from the first playable build —
budgets are cheap to hold from the start and near-impossible to claw back
after a year of unowned accretion. Mobile and VR (where missed frames cause
discomfort) demand the strictest form.

#### When not to use
Pure turn-based/UI-driven games without continuous animation can relax to
responsiveness targets ([`06`](06-quality-attributes-tradeoffs.md) latency
framing) — though load-time and battery budgets still apply.

#### Decision criteria
- Target platform floor (weakest supported device), target rate (30/60/120,
  VR ≥ 90), and thermal envelope define the budget — derive it, don't assume
  16.6 ms.
- If any subsystem has no measured cost, the budget isn't real yet.
- If the worst 1% of frames is more than ~2× the median, chase spikes
  (allocation/GC, synchronous loads §16, pathological algorithms) before
  shaving averages.

#### Common mistakes
- Optimizing based on averages while players complain about hitches the
  average hides.
- Profiling debug builds or only on high-end developer hardware.
- Ignoring sustained thermal behavior on mobile — the two-minute benchmark
  passes, the twenty-minute session throttles.
- Having no degradation levers, so the only response to a blown budget late
  in development is cutting features under pressure.
- Death by a thousand cuts with no budget ownership — every individual
  feature was "cheap."

#### Related patterns
Performance patterns (§14) — the standard fixes; asset streaming (§16) —
load-hitch avoidance; fixed-tick pipeline (§2) — decoupling sim cost from
render cost; the general performance-efficiency NFR in
[`06` §2](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog).

#### Sources
Robert Nystrom, *Game Programming Patterns* (optimization chapters' framing);
industry profiling practice as documented in engine profiler/tracing
documentation and GDC performance postmortems; frame-pacing guidance from
platform vendor documentation.

---

## 16. Asset Pipeline & Content Streaming

#### Summary
Do asset work (format conversion, compression, atlasing, baking) **at import/
build time** in a deterministic pipeline, and load the results at runtime
**asynchronously and within explicit memory budgets** — streaming content in
and out around the player rather than loading everything up front or, worse,
synchronously mid-frame.

#### Problem it addresses
Assets dominate a game's size and memory footprint. Without a pipeline,
runtime pays for work that could have been precomputed (decoding, conversion,
mip generation); without budgets, content growth silently exceeds the memory
ceiling of the weakest target device; without async loading, every load is a
frame hitch (§15) or a loading screen.

#### Description / How it works
- **Import-time vs runtime split:** source assets (PSD, FBX/glTF, WAV) are
  transformed once, deterministically, into runtime-optimal formats
  (compressed textures, platform audio formats, baked lighting/navmeshes) by
  a rebuildable pipeline — cached, versioned, and reproducible from sources,
  exactly like a code build ([`07`](07-security-reliability-operations.md)
  supply-chain reasoning applies to asset pipelines too).
- **Memory budgets per category:** textures, meshes, audio, UI each get an
  explicit budget per target platform, tracked by tooling, so content
  creators discover "over budget" at authoring time, not at device-QA time.
- **Async loading & streaming:** load off the critical path (background
  threads/async IO), reference-count or scene-scope asset lifetimes, and
  stream in/out by proximity, level section, or LOD tier. Synchronous loads
  are reserved for loading screens.
- **Load-time architecture:** startup loads the minimum to reach
  interactivity (the game-side analog of web LCP thinking,
  [`04` §4](04-web-application-design.md#4-frontend-performance--core-web-vitals));
  everything else defers, preloads predictively (next area, likely-needed
  effects), or streams on demand — with designed placeholder behavior
  (low-res first, hidden until ready) instead of pop-in surprises.
- **Stable references:** gameplay references content by stable ID through an
  indirection (asset database/registry), not by file path — the same
  stable-ID rule as §8/§11, and the enabler for patching and DLC.

#### Benefits
- Runtime does no work a build machine could have done — faster loads, lower
  memory, better battery.
- Budgets convert "the game crashes on 4 GB devices" into an authoring-time
  lint error.
- Async/streaming removes the hitch class of frame spikes (§15) and unlocks
  larger-than-memory worlds.

#### Costs & trade-offs
- The pipeline is real infrastructure: import tooling, caching, versioning,
  and invalidation bugs ("why is this texture stale?") are an ongoing cost.
- Streaming adds failure modes single-load games lack: pop-in, placeholder
  visibility, ordering bugs (using an asset before it's resident), and
  IO-bandwidth contention.
- Predictive preloading trades memory for smoothness — the budget must
  reserve headroom for it.

#### When to use
Scales with content volume: every game benefits from the import-time/runtime
split and stable IDs; explicit budgets become mandatory when targeting memory-
constrained devices; streaming becomes mandatory when content exceeds memory
or load-screen tolerance.

#### When not to use
A small game whose entire content set fits comfortably in the weakest target's
memory can load everything at startup and skip streaming complexity entirely —
the budgets are still worth writing down, the streaming machinery is not.

#### Decision criteria
- Does total resident content exceed the weakest device's budget? Streaming
  required; otherwise optional.
- Is time-to-interactive above target? Split startup loading into
  minimum-to-play + deferred.
- Any synchronous IO reachable during gameplay? That's a latent hitch —
  make it async or preloaded.

#### Common mistakes
- Doing import work at runtime "temporarily" — decode/convert costs shipping
  to players as load time and battery drain.
- No per-category budgets, discovering memory ceilings on real devices during
  certification/release QA.
- Streaming systems without designed placeholder states — pop-in becomes the
  aesthetic.
- Referencing assets by path from gameplay code, making every content
  reorganization a code change and breaking patchability (§8's stable-ID rule).

#### Related patterns
Data-driven content (§8) — the data half of the same pipeline; save versioning
(§11) — stable IDs; frame budget (§15) — hitch avoidance; supply-chain and
build reproducibility reasoning in [`07`](07-security-reliability-operations.md).

#### Sources
Jason Gregory, *Game Engine Architecture* (resource and asset-pipeline
chapters); engine asset-import/streaming documentation across major engines
(the pattern is uniform even where mechanisms differ).

---

## 17. Testing & Verification for Games

#### Summary
Games are testable to a degree folklore denies — *if* the architecture
cooperates: a headless deterministic sim core (§1, §4) enables **golden-replay
regression tests**, **invariant/property tests over fuzzed command streams**,
**statistical balance tests**, and **autoplay agents**, alongside conventional
unit tests for pure logic.

#### Problem it addresses
"Games can't be tested automatically" usually means "we entangled simulation
with rendering, so nothing runs headless." The result is regression-by-patch:
every balance change or refactor risks silently breaking combat math, replay
compatibility, save migration, or determinism, and only manual playtesting —
expensive, slow, unrepeatable — stands in the way.

#### Description / How it works
- **Golden-replay regression:** record (seed + command log + final state
  checksum) for a corpus of representative sessions (§3); CI re-runs each
  replay headless and compares checksums. Any unintended change to simulation
  behavior fails loudly, with the diverging tick identified (§4 rule 5).
  Intended balance changes update the goldens explicitly — making "this patch
  changes outcomes" a reviewed artifact.
- **Determinism tests:** run the same (seed, commands) twice — and on every
  target platform/build configuration — and require bit-identical state.
  This single test enforces all of §4 mechanically.
- **Command fuzzing / property tests:** generate random-but-valid command
  streams and assert invariants that must hold in *any* game (HP within
  bounds, no negative currency, no orphaned entity references, conservation
  rules) rather than specific outcomes. Finds the states designers never
  reach by hand.
- **Statistical balance tests:** run thousands of headless fights/generations
  across seeds and assert distribution properties (win rates within bands,
  drop rates near design targets, difficulty curve monotonicity). Catches
  balance regressions as numbers, not player complaints.
- **Autoplay agents:** scripted or AI-driven bots playing the real game loop
  (headless or full build) for soak testing, progression validation ("is the
  tutorial completable?"), and tuning data at pre-launch scale.
- **Save/migration corpus tests:** retained real saves from every shipped
  version, loaded and migrated in CI (§11).
- **Layer separation in testing:** presentation gets thin smoke tests
  (boots, renders, responds); the heavy investment goes where headless
  execution makes tests fast and deterministic — the sim core.

#### Benefits
- Regression safety for the two things players punish hardest: broken saves
  and stealth-changed game balance.
- The same harness is a *design* tool: balance simulation at content-authoring
  time (§8) reuses the test infrastructure.
- Determinism tests convert §4 from a convention into an enforced property.

#### Costs & trade-offs
- All of it is gated on §1/§4 architecture — with an entangled sim, the
  cost of testing is the cost of the retrofit rewrite.
- Golden corpora and save corpora need curation; over-broad goldens make
  every intended change a noisy mass-update.
- Statistical tests need thoughtful bands — too tight flakes on variance, too
  loose misses regressions.
- Fun, feel, and clarity remain human judgments — automation frees
  playtesting *for* those, it does not replace it.

#### When to use
The sim-core test set (goldens, determinism, invariants) belongs to any game
meeting §1's conditions — it is the payoff for that architecture. Statistical
and autoplay layers earn their cost as content volume and balance surface
grow.

#### When not to use
A game-jam prototype tests nothing and that's correct. Presentation-heavy,
logic-light games (narrative/visual) get more value from smoke tests and
save-corpus tests than from simulation harnesses they don't need.

#### Decision criteria
- Can the sim run headless? If not, fix that first (§1) — nothing else here
  is reachable.
- Does the game have PvP verification or replay features? Then determinism
  tests are mandatory, not optional (§12).
- Is balance a live, evolving surface? Statistical tests and autoplay agents
  move from luxury to core tooling.

#### Common mistakes
- Testing through the UI (pixel/screen-driven end-to-end tests) as the
  primary strategy — slow, flaky, and coupled to presentation churn; the
  stable interface is the command/event boundary (§1, §3).
- Golden replays with no policy for intended changes, training the team to
  rubber-stamp mass-updates and thereby miss real regressions.
- Running determinism tests on one platform only — cross-platform divergence
  (§4 rule 4) is precisely what single-platform CI cannot see.
- Letting the test harness use a different code path than shipping ("test
  mode") — the harness must drive the same kernel players run.

#### Related patterns
Simulation/presentation separation (§1) — the enabling architecture;
determinism (§4) — the enforced property; command pattern (§3) — the test
input format; save versioning (§11) — the corpus tests; general test-pyramid
reasoning in [`07`](07-security-reliability-operations.md).

#### Sources
Malte Skarupke, "Automated AI Testing: Simple tests will save you time"
(*Game AI Pro*, online edition 2021); Igor Borovikov et al., "AI-Driven
Autoplay Agents for Prelaunch Game Tuning" (*Game AI Pro*, online edition
2021); golden-master/property-based testing practice applied to deterministic
simulations.

---

## 18. Game-Specific Quality Attributes

The general NFR catalog in [`06` §2](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog)
applies to games too (performance, reliability, security, usability). Games add
attributes not prominent in that general catalog:

| Attribute | What it means for a game | Primary tension |
|---|---|---|
| **Determinism** | Same inputs + same seed → bit-identical output, every time, on every machine/path (live, offline-calc, replay, PvP verification) — see §4 | vs. floating-point/platform convenience — non-deterministic shortcuts are often *individually* harmless and only fail in combination |
| **Frame-time budget** | A hard per-frame deadline (e.g. 16.6 ms at 60 Hz), not an average-latency SLO — see §15 | vs. feature richness — every added per-frame system competes for the same fixed budget |
| **Replayability** | A stored input/command log can reproduce an exact past session | vs. content-update velocity — a format/rule change can invalidate old replays without an explicit versioning discipline |
| **Load time / startup** | Time to interactive content, especially on mobile (analogous to Core Web Vitals' LCP, [`04` §4](04-web-application-design.md#4-frontend-performance--core-web-vitals)) — see §16 | vs. asset richness/upfront content loading |
| **Mobile memory/thermal ceiling** | A hard, non-negotiable ceiling with no swap headroom; sustained (not just burst) load causes thermal throttling | vs. simultaneous on-screen complexity (a full-party, full-effects combat scene) |
| **Fairness (PvP integrity)** | An async or live PvP result must be independently verifiable, not just client-reported (§12, §13) | vs. client-authoritative convenience (trusting the client is simpler, but not fair-verifiable) |
| **Content/data velocity** | New content ships as data, not code, and doesn't corrupt existing saves/replays (§8, §11) | vs. mechanic novelty — content that doesn't fit the existing schema needs new interpreter code |
| **Save durability** | Player progress survives crashes, updates, and device changes (§11) | vs. schema/refactor freedom — every save-touching change carries a migration cost |

**Decision guidance:** for designs where offline-progress calculation or
verifiable PvP are core pillars, **determinism and fairness are not optional
trade-offs to weigh against convenience** — they are prerequisites the other
quality attributes must be achieved *within*, the same way security
correctness is non-negotiable in [`06`](06-quality-attributes-tradeoffs.md)'s
general framing. Treat them like a safety constraint, not a tunable NFR.

> **Standards note:** these game-specific attributes map onto the general **ISO/IEC 25010** quality model ([06 §2](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog)) as domain-specific instances rather than new characteristics: frame-time budget and load time fall under **Performance Efficiency**; save durability and replayability fall under **Reliability**; fairness/PvP integrity falls under **Security**; and determinism, taken to its logical extreme (a physics/combat sim whose divergence could mean a corrupted competitive result or an exploitable desync), is this domain's clearest instance of 25010's **Safety** characteristic ([06 §2](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog)) — a fail-safe/deterministic-by-construction requirement, not a tunable preference. See [`13`](13-standards-crosswalk.md) for the full crosswalk.

---

## 19. Anti-patterns

| Anti-pattern | Why it's harmful | Avoid by |
|---|---|---|
| **Retrofitted determinism** | Non-deterministic habits (unseeded RNG, unordered iteration, render-tick-dependent math) accumulate silently until offline-calc/replay/PvP results diverge from live play, at which point fixing it is a rewrite of the combat kernel, not a patch | Enforce determinism (§4) from the first line of simulation code, verified by tests (§17) |
| **Simulation entangled with rendering** | Blocks headless offline calc, server-side PvP verification, and automated testing | Sim/presentation separation (§1) as a hard boundary, not a convention |
| **God update loop** | One monolithic per-frame update callback doing physics, AI, UI, and rendering inline becomes unreadable and impossible to budget per-subsystem | Fixed-tick pipeline (§2) with explicit per-subsystem budgets (§15) |
| **Undocumented simultaneous-event resolution order** | Two events that could resolve in either order (e.g. mutual-kill) silently produce different, unreproducible results depending on incidental iteration order | Deterministic update ordering (§4), documented as part of the combat-kernel contract |
| **Client-authoritative PvP** | A client-reported result with no independent re-simulation or server validation is trivially exploitable | Independently-verifiable deterministic replay (§12) or server-authoritative netcode (§13) |
| **Content-as-code for high-volume categories** | Every new item/skill/affix requires a code change and rebuild, throttling content velocity and blocking moddability | Data-driven content pipeline (§8) against versioned schemas |
| **Executable mod/import data** | Loading untrusted external content as anything that can reference or execute code is a security hole disguised as a content feature | Restrict external/mod data to plain, schema-validated, non-executable formats (§8, §11) |
| **Boolean-flag state explosion** | Independent flags for what are really mutually-exclusive modes admit invalid combinations and become unreadable as they accumulate | State machines (§6) for genuinely mutually-exclusive modes; data (status-effect lists) for genuinely orthogonal, simultaneous concerns |
| **Save format as serialized runtime objects** | Couples save compatibility to internal refactors and engine versions; often can embed executable references | Explicit, versioned save schema with migrations (§11) |
| **Premature performance architecture** | Building custom ECS/pooling/partitioning machinery before any measurement shows the need adds bug surface without benefit | Profile first (§15); apply §14 patterns where evidence points |

---

## Sources

Glenn Fiedler, "Fix Your Timestep!," "Deterministic Lockstep," "Snapshot
Interpolation," "Floating Point Determinism" — https://gafferongames.com/.
Robert Nystrom, *Game Programming Patterns* —
https://gameprogrammingpatterns.com/ (Game Loop, Update Method, Command, State,
Observer, Event Queue, Component, Type Object, Double Buffer, Object Pool,
Spatial Partition, Data Locality, Dirty Flag). Gabriel Gambetta, *Fast-Paced
Multiplayer* series — https://www.gabrielgambetta.com/client-server-game-architecture.html.
GGPO rollback SDK (Tony Cannon) — https://github.com/pond3r/ggpo. Sander
Mertens, *ECS FAQ* — https://github.com/SanderMertens/ecs-faq. Adam Martin,
"Entity Systems are the future of MMOG development." Scott Bilas, "A
Data-Driven Game Object System" (GDC 2002). Damian Isla, "Handling Complexity
in the Halo 2 AI" (GDC 2005). Blizzard, "Overwatch Gameplay Architecture and
Netcode" (GDC 2017). *Game AI Pro* series (free chapters) —
http://www.gameaipro.com/. Richard Fabian, *Data-Oriented Design* —
https://www.dataorienteddesign.com/dodbook/. Mike Acton, "Data-Oriented Design
and C++" (CppCon 2014). Jason Gregory, *Game Engine Architecture*. General
event-driven-architecture and event-sourcing sources shared with
[`02`](02-architecture-patterns.md) and [`09`](09-references.md).

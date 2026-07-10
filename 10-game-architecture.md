# Game Architecture

Real-time and turn-based games share an architectural concern general application
design rarely centers: a **simulation** that must run correctly, repeatably, and
fast, decoupled from however it is displayed. This file covers the patterns that
follow from that — sim/presentation separation, fixed-tick update pipelines, the
command pattern for deterministic replay, entity-modeling choices (ECS vs
scene-graph/node composition), game state machines, event-driven communication,
data-driven content pipelines, and the netcode patterns that extend a
deterministic simulation to multiplayer — plus the quality attributes a game adds
to the general catalog in [`06`](06-quality-attributes-tradeoffs.md).

This file is **technology- and engine-agnostic**, consistent with the rest of
this reference. Concrete engine mechanics (e.g. Godot's node lifecycle,
GDScript's threading model) live in the Coding library's
[`languages/gdscript.md`](../Coding/languages/gdscript.md) and
[`13-game-runtime-and-determinism.md`](../Coding/13-game-runtime-and-determinism.md)
— this file covers the structural decisions that hold regardless of engine.

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
it later is a rewrite, not a patch (see §9).

#### Decision criteria
- Does any feature require the simulation to run without rendering (offline
  calc, server verification, automated testing)? If yes, this is required, not
  optional.
- Does the simulation need to be reproduced exactly later (replay, "what would
  have happened")? If yes, the event log and determinism (§8, and the Coding
  library's [determinism doc](../Coding/13-game-runtime-and-determinism.md))
  are load-bearing together.

#### Common mistakes
- Simulation code reading a rendered sprite's transform, an animation's
  progress, or any other presentation-layer value to make a gameplay decision
  — this silently couples simulation correctness to frame timing.
- Presentation re-deriving "what happened" from a before/after state diff
  instead of consuming explicit events — ambiguous when multiple effects
  resolve in the same tick (see §3).
- Treating the separation as a one-time refactor rather than an enforced
  boundary — a single addition of a render-dependent shortcut months later
  reintroduces the coupling.

#### Related patterns
Hexagonal Architecture / Ports & Adapters ([`02` §3.3](02-architecture-patterns.md#33-hexagonal-architecture-ports--adapters))
— the simulation core is the "domain," presentation is a driven adapter. Command
pattern (§3). Fixed-tick pipeline (§2).

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
  hardware speed — a prerequisite for determinism (§8).
- Decouples "how often do we simulate" from "how often do we render," so each
  can be tuned independently for correctness vs smoothness/battery.
- A stalled or dropped frame doesn't desynchronize game time from real time.

#### Costs & trade-offs
- Requires explicit interpolation/extrapolation work in the render layer to
  avoid visible stepping at low tick rates.
- An unbounded catch-up loop after a long stall ("spiral of death") can itself
  blow the frame budget — must be capped, with large gaps handled as an
  explicit batch pass instead (see §5, offline progress).

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
state).

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
                          actor: hero_3,           tick order
                          skill: "Fireball",
                          target: cell(4,2)}
```
Every action a hero, an enemy AI, or a scripted Tactics/automation rule wants
to take is expressed as a command: a small, serializable value describing the
*intent* (who, what, where) not the *effect*. The simulation core consumes an
ordered queue of commands per tick and applies them deterministically. A full
match/session replay is then just the recorded command list plus the initial
seed — re-running the same commands through the same simulation reproduces the
same result exactly (given the determinism practices in §8).

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
concrete mechanism that makes §1 and §8 actually implementable, not just
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
commands specifically; simulation/presentation separation (§1); deterministic
update ordering (§8).

#### Sources
Robert Nystrom, *Game Programming Patterns* — Command; Glenn Fiedler,
"Deterministic Lockstep."

---

## 4. Entity Modeling: ECS vs Scene-Graph/Node Composition

#### Summary
Two dominant ways to structure game objects: **Entity-Component-System (ECS)** —
data-oriented, entities are IDs with attached component data, systems operate
over component arrays — versus **scene-graph/node composition** — object-
oriented, each game object is a tree of composed nodes/components with
behavior attached directly.

#### Problem it addresses
Game objects need both varied behavior (a hero has stats, skills, status
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
ECS data model.

#### Benefits

*ECS:* Excellent cache locality and bulk-iteration performance at high entity
counts (thousands+); trivially easy to add a new cross-cutting system without
touching entity definitions; naturally data-oriented, which pairs well with
data-driven content (§6) and with deterministic, order-explicit iteration (the
Coding library's [determinism doc](../Coding/13-game-runtime-and-determinism.md)
§10 requires exactly this kind of explicit, stable iteration order).

*Scene-graph/node composition:* Matches most game engines' native authoring
tools directly (visual scene editors, inspector-driven composition), so
designers can assemble variants without code; simpler mental model for small-
to-moderate entity counts; less architectural machinery to build/maintain.

#### Costs & trade-offs

*ECS:* More upfront architecture (component storage, system scheduling); less
natural fit with an engine's built-in scene editor/inspector workflow unless
the engine has first-class ECS support; can feel like over-engineering below
the entity-count threshold where its performance benefit matters.

*Scene-graph/node composition:* Node-per-entity overhead (tree traversal,
per-node dispatch) scales worse at very high entity counts than tightly-packed
component arrays; behavior can still creep toward inheritance-chain rigidity
if composition discipline lapses.

#### When to use

*ECS:* Large entity counts with performance-critical bulk updates (large-scale
battles, particle-like swarms, colony/city simulations); when the same set of
cross-cutting systems (movement, damage, status) must apply uniformly across
very different entity kinds.

*Scene-graph/node composition:* Small-to-moderate entity counts (a squad-based
combat game with a handful of heroes and a bounded per-fight enemy count is
solidly in this range); when leveraging the engine's native editor/authoring
tools for composition is valuable; when team size/expertise favors the
engine's default object model over building/learning a separate ECS layer.

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
  the Coding library's [determinism doc](../Coding/13-game-runtime-and-determinism.md) §10.

#### Related patterns
Composition over inheritance ([`03` §2.6](03-software-design-principles.md#26-composition-over-inheritance));
data-driven content pipeline (§6); deterministic update ordering (external:
Coding library §10).

#### Sources
Robert Nystrom, *Game Programming Patterns* — Component; general ECS
literature (Adam Martin, "Entity Systems are the future of MMOG development");
Sandi Metz on composition/inheritance trade-offs.

---

## 5. State Machines for Game Logic

#### Summary
Model discrete-mode game logic — combat phases, AI behavior, UI flow, a
character's action state — as explicit **finite state machines (FSMs)** or
**hierarchical/behavior-tree variants**, rather than networks of boolean flags.

#### Problem it addresses
"Is the hero currently casting, stunned, dead, or channeling?" answered by a
scatter of independent booleans (`is_casting`, `is_stunned`, `is_dead`,
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
flexible branching logic than a flat FSM handles well.

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
logic: combat action state, AI behavior, UI screen flow, an automation/Tactics
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
conditions" (AI) than "which single mode am I in"? Favor a behavior tree over a
flat FSM.

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
pattern (GoF); behavior trees (AI-specific generalization).

#### Sources
Robert Nystrom, *Game Programming Patterns* — State; GoF *Design Patterns* —
State pattern; behavior-tree literature (Damian Isla, "Handling Complexity in
the Halo 2 AI").

---

## 6. Data-Driven Content Pipeline

#### Summary
Author game content — classes, skills, items, affixes, enemies, level/rift
modifiers — as **external data**, loaded and interpreted by generic code,
rather than encoding each piece of content as bespoke source code.

#### Problem it addresses
A game whose content scales primarily through variety and volume (many item
affixes, many skills, procedurally-recruited hero traits, endlessly-scaling
rift modifiers) cannot sustainably require a code change and a rebuild for
every new piece of content — content velocity and, for this project
specifically, moddability both depend on content living outside compiled code.

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
skills, affixes, enemies, procedurally-generated content, rift/dungeon
modifiers) — which describes essentially every content axis in a deep-
itemization, procedurally-recruited-hero design.

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
  plain, schema-validated, non-executable formats (see the Coding library's
  [GDScript security rules](../Coding/languages/gdscript.md#12-security) for
  the concrete engine-level version of this).
- No schema versioning — a content-format change breaks every existing data
  file with no migration path, and breaks save compatibility for players whose
  save references now-invalid content IDs.
- Business logic creeping into "data" as ad hoc scripting/expression strings
  evaluated unsafely, effectively becoming an unaudited second scripting
  language.

#### Related patterns
Configuration over code (general software design); the Coding library's
[GDScript `Resource`-as-data pattern](../Coding/languages/gdscript.md#7-exports-resources--scene-composition)
for the concrete Godot mechanism; ECS's component-as-data model (§4) pairs
naturally with data-driven content.

#### Sources
General data-driven-design literature in game development; Robert Nystrom,
*Game Programming Patterns* — Type Object.

---

## 7. Communication: Events and State Ownership in Games

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
  answer within the same tick (see §8's ordering requirements) — an
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

## 8. Netcode Patterns for Asynchronous / Replay-Based Multiplayer

#### Summary
For multiplayer built on **asynchronous, snapshot/replay-based** interaction
(one player's squad fights a snapshot of another's, rather than both playing
live simultaneously) — as opposed to real-time lockstep or client-server
live-action netcode — the architecture is closer to "replay the deterministic
simulation with a captured opponent input set" than to traditional real-time
networking.

#### Problem it addresses
Live real-time multiplayer (lockstep, rollback, authoritative-server tick
sync) solves a much harder problem than this project needs, at much higher
cost: async PvP only requires that a fight between a live player's current
build and a *stored snapshot* of another player's build produces a fair,
reproducible, tamper-resistant result — no live connection between the two
players is ever required.

#### Description / How it works
```
Player A's squad (live)          Player B's squad (snapshot, stored server-side)
         │                                    │
         └──────────────┬─────────────────────┘
                         ▼
         Deterministic simulation core (§1)
         runs once, using both squads' data
         as input, with a fixed/shared seed
                         │
                         ▼
              Result + full event log
     (verifiable by re-running the same simulation
      with the same inputs, server-side or client-side)
```
Each player's squad/build (heroes, gear, stats, Tactics/automation rules) is
serialized as a snapshot. An async PvP fight is a single deterministic
simulation run (§1, §2, §3) over both snapshots — computed once, its result is
final and doesn't require either player to be online simultaneously.
**Fairness and anti-cheat both reduce to determinism**: because the simulation
is deterministic and both squads' data are captured/stored, any party
(server, opponent's client, a third-party auditor) can independently
re-run the exact same simulation and must get the exact same result — a
mismatch is proof of tampering or a non-deterministic bug, not a legitimate
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
  different (lockstep/rollback) netcode layer.
- Snapshot staleness is a design question: if Player B improves their squad
  after being snapshotted, defenders may be fighting an out-of-date version of
  themselves — needs an explicit policy (refresh cadence, "defense squad" the
  player actively curates, etc.), not just an implementation detail.
- Full trust in the determinism requirement (§1, §2, §3, and the Coding
  library's [determinism doc](../Coding/13-game-runtime-and-determinism.md))
  — any non-determinism bug becomes a fairness/anti-cheat hole, not just a
  visual glitch.

#### When to use
Async PvP (arena/defense-squad formats), leaderboard-driven competition,
and any "my build vs. a stored snapshot of your build" interaction model —
exactly this project's chosen PvP shape.

#### When not to use
Live, simultaneous, latency-sensitive multiplayer (real-time co-op, live PvP
where both players act and react to each other in the moment) needs
lockstep/rollback/authoritative-server netcode instead — a fundamentally
different, more expensive architecture this pattern does not cover.

#### Decision criteria
Do both participants need to be online and reacting to each other live? If
yes, this pattern doesn't apply — need real-time netcode. Is the interaction
"my current build vs. a captured snapshot," resolved as a single deterministic
computation? This pattern is the direct, low-cost fit.

#### Common mistakes
- Treating the client-computed result as authoritative without an independent
  re-simulation to verify it — reopens exactly the cheating vector determinism
  was meant to close.
- No defined snapshot-refresh/staleness policy, leading to confusing or unfair
  "defense" outcomes players can't reason about.
- Allowing any non-deterministic element (wall-clock-seeded RNG, unordered
  iteration, floating-point order-dependence) into the combat kernel — this
  pattern has zero tolerance for the anti-patterns catalogued in the Coding
  library's [determinism doc](../Coding/13-game-runtime-and-determinism.md) §14.

#### Related patterns
Command pattern (§3) — the replay mechanism; simulation/presentation
separation (§1); the general distributed-systems idempotency/verification
reasoning in [`02` §7.7](02-architecture-patterns.md#77-idempotency).

#### Sources
Glenn Fiedler, "Deterministic Lockstep" (the general determinism-for-
multiplayer reasoning, applied here to the async/snapshot case specifically);
general async-PvP/"defense squad" pattern as used across mobile idle/RPG
games.

---

## 9. Game-Specific Quality Attributes

The general NFR catalog in [`06` §2](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog)
applies to games too (performance, reliability, security, usability). Games add
attributes not prominent in that general catalog:

| Attribute | What it means for a game | Primary tension |
|---|---|---|
| **Determinism** | Same inputs + same seed → bit-identical output, every time, on every machine/path (live, offline-calc, replay, PvP verification) | vs. floating-point/platform convenience — non-deterministic shortcuts are often *individually* harmless and only fail in combination |
| **Frame-time budget** | A hard per-frame deadline (e.g. 16.6 ms at 60 Hz), not an average-latency SLO | vs. feature richness — every added per-frame system competes for the same fixed budget |
| **Replayability** | A stored input/command log can reproduce an exact past session | vs. content-update velocity — a format/rule change can invalidate old replays without an explicit versioning discipline |
| **Load time / startup** | Time to interactive content, especially on mobile (analogous to Core Web Vitals' LCP, [`04` §4](04-web-application-design.md#4-frontend-performance--core-web-vitals)) | vs. asset richness/upfront content loading |
| **Mobile memory/thermal ceiling** | A hard, non-negotiable ceiling with no swap headroom; sustained (not just burst) load causes thermal throttling | vs. simultaneous on-screen complexity (a full-squad, full-effects combat scene) |
| **Fairness (PvP integrity)** | An async or live PvP result must be independently verifiable, not just client-reported | vs. client-authoritative convenience (trusting the client is simpler, but not fair-verifiable) |
| **Content/data velocity** | New content ships as data, not code, and doesn't corrupt existing saves/replays | vs. mechanic novelty — content that doesn't fit the existing schema needs new interpreter code |

**Decision guidance:** for a design with offline-progress calculation and
async PvP as core pillars (as in this project), **determinism and fairness are
not optional trade-offs to weigh against convenience** — they are prerequisites
the other quality attributes must be achieved *within*, the same way security
correctness is non-negotiable in [`06`](06-quality-attributes-tradeoffs.md)'s
general framing. Treat them like a safety constraint, not a tunable NFR.

---

## 10. Anti-patterns

| Anti-pattern | Why it's harmful | Avoid by |
|---|---|---|
| **Retrofitted determinism** | Non-deterministic habits (unseeded RNG, unordered iteration, render-tick-dependent math) accumulate silently until offline-calc/replay/PvP results diverge from live play, at which point fixing it is a rewrite of the combat kernel, not a patch | Enforce determinism (§8, external Coding-library rules) from the first line of simulation code |
| **Simulation entangled with rendering** | Blocks headless offline calc, server-side PvP verification, and automated testing | Sim/presentation separation (§1) as a hard boundary, not a convention |
| **God update loop** | One `_process`/`update()` doing physics, AI, UI, and rendering inline becomes unreadable and impossible to budget per-subsystem | Fixed-tick pipeline (§2) with explicit per-subsystem budgets |
| **Undocumented simultaneous-event resolution order** | Two events that could resolve in either order (e.g. mutual-kill) silently produce different, unreproducible results depending on incidental iteration order | Deterministic update ordering, documented as part of the combat-kernel contract (external Coding-library §10) |
| **Client-authoritative PvP** | A client-reported result with no independent re-simulation is trivially exploitable | Netcode via independently-verifiable deterministic replay (§8) |
| **Content-as-code for high-volume categories** | Every new item/skill/affix requires a code change and rebuild, throttling content velocity and blocking moddability | Data-driven content pipeline (§6) against versioned schemas |
| **Executable mod/import data** | Loading untrusted external content as anything that can reference or execute code is a security hole disguised as a content feature | Restrict external/mod data to plain, schema-validated, non-executable formats |
| **Boolean-flag state explosion** | Independent flags for what are really mutually-exclusive modes admit invalid combinations and become unreadable as they accumulate | State machines (§5) for genuinely mutually-exclusive modes; data (status-effect lists) for genuinely orthogonal, simultaneous concerns |

---

## Sources

Glenn Fiedler, "Fix Your Timestep!," "Deterministic Lockstep," "Floating Point
Determinism" — https://gafferongames.com/. Robert Nystrom, *Game Programming
Patterns* — https://gameprogrammingpatterns.com/ (Game Loop, Update Method,
Command, State, Observer, Event Queue, Component, Type Object, Double Buffer).
Adam Martin, "Entity Systems are the future of MMOG development." Damian Isla,
"Handling Complexity in the Halo 2 AI" (GDC). General event-driven-architecture
and event-sourcing sources shared with [`02`](02-architecture-patterns.md) and
[`09`](09-references.md).

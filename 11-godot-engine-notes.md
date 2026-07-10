# Godot Engine Notes

> **Scope & time-sensitivity.** Unlike the rest of this reference, this file is
> deliberately **engine-specific**: it maps the technology-agnostic patterns of
> [`10-game-architecture.md`](10-game-architecture.md) onto the Godot engine.
> Engine facts here were **verified against Godot 4.7 (stable) in July 2026**
> and will age; re-verify against the [official documentation](https://docs.godotengine.org/en/stable/)
> before relying on version-specific claims. The *reasoning* — which trade-off
> each mechanism resolves — is the durable part.

Each section names the general pattern it implements, so this file reads as an
appendix to [`10`](10-game-architecture.md), not a substitute for it.

---

## 1. Scenes, Nodes, and Composition

*Implements: entity modeling via node composition ([`10` §5](10-game-architecture.md#5-entity-modeling-ecs-vs-scene-graphnode-composition)); composition over inheritance ([`03` §2.6](03-software-design-principles.md#26-composition-over-inheritance)).*

Godot's core model is a **tree of nodes**; a **scene** is a saved, instantiable
subtree, and scenes nest freely (there is no separate prefab concept — a scene
is both). This is an *entity-component* style (nodes carry data **and**
behavior), not an ECS — composition happens at a higher level than in
data-oriented ECS designs, trading bulk-iteration performance for editor
ergonomics and explicitness. Godot's own architecture writeup is candid about
this trade: engine-internal heavy lifting lives in data-oriented *servers*
(§4), while the node tree is a high-level interface to them.

Practical rules from the official best-practices docs:

- **Prefer scene composition over inheritance chains.** Build variants by
  composing child scenes/nodes and exporting configuration, not by deep
  `extends` hierarchies — the same reasoning as [`10` §5](10-game-architecture.md#5-entity-modeling-ecs-vs-scene-graphnode-composition)'s
  warning against inheritance regrowth.
- **"Signal up, call down."** A node may call methods on its own children
  (it owns them); it must not reach *up* or *across* the tree — instead it
  emits signals ancestors/siblings connect to. This keeps scenes
  self-contained and reusable, and is the in-engine form of
  [`10` §10](10-game-architecture.md#10-communication-events-and-state-ownership-in-games)'s
  event-driven decoupling.
- **Scenes vs scripts:** a script adds behavior to one node type; a scene
  packages a configured subtree. Reusable *behavior* → script (possibly with
  `class_name`); reusable *assembly* → scene.
- **Autoloads sparingly.** Autoloads (singletons) are appropriate for genuinely
  global, self-owned systems (a quest system, a scene switcher); they are the
  engine's global-state hazard otherwise — the official docs walk through why
  a global "sound manager" style autoload harms debuggability and resource
  scoping, and since GDScript's `static var`/`static func`, shared helpers no
  longer need an autoload at all. Same trade-offs as any singleton:
  convenient, globally coupled.

---

## 2. Update Pipeline: `_process`, `_physics_process`, and Physics Interpolation

*Implements: fixed-tick pipeline ([`10` §2](10-game-architecture.md#2-fixed-tick-update-pipeline)).*

Godot ships the fixed-tick pattern natively:

- `_physics_process(delta)` runs at a **fixed tick rate** — project setting
  `physics/common/physics_ticks_per_second`, default **60** — independent of
  rendered frame rate. Gameplay-affecting logic belongs here
  ([`10` §2](10-game-architecture.md#2-fixed-tick-update-pipeline)'s decision
  criterion: anything feeding game state → fixed tick).
- `_process(delta)` runs once per rendered frame at variable rate — reserve it
  for presentation (cosmetic animation, camera smoothing, UI).
- **Physics interpolation** (2D and 3D, built-in, off by default) renders
  objects interpolated between the previous and current physics tick —
  exactly the Fiedler "Fix Your Timestep!" mechanism, including its documented
  trade-off: the display shows the state *slightly in the past*, a small input-
  latency cost. `Engine.get_physics_interpolation_fraction()` exposes the
  blend factor for custom interpolation.
- The docs explicitly note built-in interpolation may be a poor fit for
  networked games whose timing comes from a server — custom interpolation
  aligned to network ticks is often better
  ([`10` §13](10-game-architecture.md#13-real-time-netcode-models)).

**Caveat:** running gameplay in `_physics_process` gives frame-rate
*independence*, not *determinism* — see §5 below.

---

## 3. Data-Driven Content with Resources

*Implements: data-driven content pipeline ([`10` §8](10-game-architecture.md#8-data-driven-content-pipeline)); moddability with its security boundary.*

- **Custom `Resource` classes** are Godot's first-class mechanism for
  [`10` §8](10-game-architecture.md#8-data-driven-content-pipeline)'s schemas:
  define a `Resource` subclass with exported fields (a skill's targeting
  shape, effect list, numbers), author instances as `.tres` files in the
  inspector, and load them as typed data. They hot-reload in the editor and
  integrate with the inspector — the "designers assemble variants without
  code" benefit.
- **Security boundary:** Godot resource files **can embed scripts**. Loading a
  `.tres`/`.res`/scene file from an untrusted source (mods, downloaded
  content, another player's data) can execute arbitrary code. This is the
  concrete engine instance of [`10` §8](10-game-architecture.md#8-data-driven-content-pipeline)'s
  "executable mod data" anti-pattern: restrict untrusted content to plain,
  schema-validated formats (JSON via `FileAccess`, glTF via `GLTFDocument`,
  images/audio via the runtime loaders) and validate on load.
- **Distribution mechanisms:** `ProjectSettings.load_resource_pack()` mounts
  PCK/ZIP packs into the virtual filesystem (fits trusted DLC/patches);
  `ZIPReader`/`FileAccess` plus the runtime loading APIs handle
  user-generated content without the editor — the official *Runtime file
  loading and saving* guide covers formats and their caveats.
- For saves, prefer explicit serialization (e.g. JSON/binary you design) over
  saving engine objects — the save-format reasoning and the untrusted-input
  rule of [`10` §11](10-game-architecture.md#11-save-data-persistence--versioning)
  both apply; `store_var`/resource saving of live objects couples the save to
  code layout and can carry embedded scripts.

---

## 4. Scaling Entity Counts: the Godot Ladder

*Implements: entity modeling at scale ([`10` §5](10-game-architecture.md#5-entity-modeling-ecs-vs-scene-graphnode-composition)); performance patterns ([`10` §14](10-game-architecture.md#14-performance-patterns-pooling-spatial-partitioning-data-locality)).*

Nodes are convenient but carry per-node overhead. Godot's documented
escalation path, in order of increasing effort:

1. **Plain nodes/scenes** — fine into the hundreds of active entities;
   engine-internal processing is already data-oriented in servers.
2. **Cheaper objects than nodes** — `RefCounted`/`Object` (or `Resource`) for
   pure-logic entities that don't need tree membership; the official "when
   and how to avoid using nodes" guidance.
3. **Clever scheduling** — time-slicing, visibility-based processing
   ([`10` §14](10-game-architecture.md#14-performance-patterns-pooling-spatial-partitioning-data-locality)'s
   logic-LOD), disabling processing on off-screen subtrees.
4. **Servers and RIDs directly** — bypass nodes and drive `RenderingServer`/
   `PhysicsServer2D/3D` with raw resource IDs; `MultiMesh` for rendering
   thousands of instances in one draw call. This is Godot's sanctioned
   "drop a level" API, not a hack.
5. **An ECS addon or GDExtension** — for genuinely large simulations, bolt a
   data-oriented layer (community ECS integrations, or custom C++/Rust via
   GDExtension) onto Godot for the sim while keeping Godot for presentation —
   which is precisely the sim/presentation split of
   [`10` §1](10-game-architecture.md#1-simulationpresentation-separation).

Godot's leadership has published the reasoning for *not* being ECS-based
(nodes as high-level interfaces over data-oriented servers); the honest
implication, matching [`10` §5](10-game-architecture.md#5-entity-modeling-ecs-vs-scene-graphnode-composition)'s
decision criteria, is that node-per-entity designs have a real ceiling and the
ladder above is the intended response.

---

## 5. Determinism in Godot

*Implements: determinism requirements ([`10` §4](10-game-architecture.md#4-determinism-requirements)); async-PvP verification ([`10` §12](10-game-architecture.md#12-netcode-patterns-for-asynchronous--replay-based-multiplayer)).*

The blunt facts:

- **Godot's built-in physics is not deterministic** — not guaranteed
  run-to-run, and certainly not cross-platform. Do not put engine physics
  inside a simulation kernel that must satisfy
  [`10` §4](10-game-architecture.md#4-determinism-requirements); use it for
  presentation and non-authoritative effects.
- **GDScript floats are 64-bit doubles** on all platforms, but the
  cross-platform floating-point caveats of [`10` §4](10-game-architecture.md#4-determinism-requirements)
  rule 4 still apply to anything crossing C#/GDExtension/platform-math
  boundaries. A deterministic kernel is safest built on **integer or
  fixed-point math** in plain GDScript/C# code you own.
- **Seeded randomness:** use owned `RandomNumberGenerator` instances with
  explicit seeds (and `state` capture) per stream — never the global `randi()`
  family inside the kernel, which presentation code can also consume
  (violating [`10` §4](10-game-architecture.md#4-determinism-requirements)
  rule 1).
- **Iteration order:** GDScript `Dictionary` preserves insertion order —
  convenient, but *insertion order is itself state*; sort keys or maintain
  explicit ordered lists for kernel iteration
  ([`10` §4](10-game-architecture.md#4-determinism-requirements) rule 2).
- A headless/server build (`--headless`, dedicated-server export) runs the
  same scripts without rendering — the natural home for
  [`10` §17](10-game-architecture.md#17-testing--verification-for-games)'s
  golden-replay CI and [`10` §12](10-game-architecture.md#12-netcode-patterns-for-asynchronous--replay-based-multiplayer)'s
  server-side re-simulation.

---

## 6. Multiplayer

*Implements: real-time netcode models ([`10` §13](10-game-architecture.md#13-real-time-netcode-models)); never-trust-the-client.*

- Godot's **high-level multiplayer API** (`MultiplayerAPI` +
  `MultiplayerPeer`) provides RPCs (`@rpc` annotation), peer management, and
  transfer modes (`reliable`, `unreliable`, `unreliable_ordered`) over
  channels, with `ENetMultiplayerPeer` (UDP), `WebSocketMultiplayerPeer`, and
  `WebRTCMultiplayerPeer` backends. `MultiplayerSpawner`/
  `MultiplayerSynchronizer` add scene replication and state sync — an
  authoritative-server/state-replication shape by default (server is peer 1,
  per-node authority assignable).
- **What it gives vs what it doesn't:** it handles transport, RPC routing,
  and replication plumbing; it does *not* implement client-side prediction,
  reconciliation, entity interpolation, lag compensation, lockstep, or
  rollback ([`10` §13](10-game-architecture.md#13-real-time-netcode-models))
  — those remain your architecture, built on top (or on the low-level
  `PacketPeer` APIs when the high-level shape doesn't fit).
- **Security:** the official docs now carry an explicit *secure multiplayer
  design* section: server-authoritative gameplay state, validate every RPC's
  arguments and rate, never trust client-reported positions/results — plus a
  built-in **authentication hook** (`SceneMultiplayer.auth_callback` /
  `complete_auth`) for verifying peers before they join. Configure RPCs as
  `authority` mode unless client calls are genuinely required
  (`any_peer` RPCs are the attack surface).
- Async-PvP designs ([`10` §12](10-game-architecture.md#12-netcode-patterns-for-asynchronous--replay-based-multiplayer))
  typically need none of this in-game: snapshots move over plain HTTPS
  (`HTTPRequest`) and verification runs headless server-side.

---

## 7. Language & Runtime Selection

*Implements: technology-selection reasoning ([`08`](08-checklists-and-templates.md) rubric applied to one engine).*

| Option | Strengths | Costs |
|---|---|---|
| **GDScript** | Tightest editor integration (hot-reload, docs, breakpoints); fast iteration; static typing (`: int`, warnings-as-errors) closes much of the safety gap and improves performance | Slowest raw execution of the three; dynamic-dispatch overhead on every scripting-API call |
| **C# (.NET)** | Much faster compute-heavy code; mature ecosystem/tooling; good for teams with existing C# skills | Heavier build/iteration loop; engine-API calls still cross the same marshalling boundary; platform support has caveats (verify current status for web/mobile targets) |
| **GDExtension (C++/Rust)** | Native performance; direct server access; the right tool for a deterministic fixed-point kernel or an ECS layer (§4, §5) | Highest complexity; per-platform builds; API surface must be registered manually |

The standard mix, mirroring [`10` §1](10-game-architecture.md#1-simulationpresentation-separation):
GDScript for presentation/glue/editor tooling, dropping to C# or GDExtension
for the simulation kernel or bulk systems *when measurement (
[`10` §15](10-game-architecture.md#15-frame-budget--profiling-discipline))
shows the need*. Godot's profiler, `--headless` mode, and per-platform
tracing hooks support that measurement.

---

## 8. Current-Version Notes (Godot 4.7, July 2026)

Time-sensitive by nature; re-verify before depending on any of these.

- **4.7 highlights:** HDR output (2D and 3D); `Control` offset transforms
  (animate UI without fighting containers); `AreaLight3D`; built-in
  `VirtualJoystick` node and controller gyro/accelerometer input; Asset Store
  replacing the Asset Library; stable Godot Android Build Environment (full
  export/publish from an Android device); Perfetto tracing as the default
  Android profiling path; `DrawableTexture2D`.
- **Accessibility:** screen-reader support (since 4.5) plus landmark
  navigation (4.7) — game UIs built from `Control` nodes inherit real
  accessibility structure; relevant to the accessibility NFRs in
  [`06`](06-quality-attributes-tradeoffs.md).
- **Recent-versions baseline** (if targeting an older 4.x): physics
  interpolation, Wayland support, and runtime FBX loading arrived in 4.3;
  typed dictionaries and embedded-game-window workflow in 4.4–4.6;
  `static var` in 4.1.
- **Migration discipline:** minor 4.x releases carry breaking changes; the
  documented per-version migration guides are the authoritative checklist
  when upgrading an existing project.

---

## Sources

Godot official documentation (verified 2026-07-10, Godot 4.7 stable):
*Best practices* (scene organization; scenes vs scripts; autoloads vs regular
nodes; when and how to avoid using nodes; data preferences), *Physics
interpolation*, *High-level multiplayer*, *Runtime file loading and saving* —
https://docs.godotengine.org/en/stable/. Juan Linietsky, "Why isn't Godot an
ECS-based game engine?" (godotengine.org blog, 2021). Godot 4.7 release notes
— https://godotengine.org/releases/4.7/. Cross-referenced against the
engine-agnostic sources in [`09-references.md`](09-references.md#game-architecture).

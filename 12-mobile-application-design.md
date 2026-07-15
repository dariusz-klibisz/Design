# Mobile Application Design & Architecture

Mobile applications differ from web and desktop applications because they run on **battery- and thermally-constrained personal devices**, under an **OS-managed lifecycle** that can suspend or kill the process at any moment, behind **app-store gatekeeping** that controls distribution and update cadence, with **intermittent connectivity** as the norm rather than the exception, and inside a **permission model** the user actively negotiates. The device is the most personal computer most users own — trust, battery, and attention are the scarce resources.

This file covers what makes mobile different, UI architecture patterns, the native-vs-cross-platform decision, framework comparisons, per-OS (iOS/Android) specifics, offline-first & sync, state/navigation/push, mobile security (OWASP MASVS/MASTG), performance, accessibility, packaging/signing/store release, mobile-specific quality attributes, and anti-patterns. It builds on [`01`](01-architecture-principles.md)–[`03`](03-software-design-principles.md), shares patterns with [`05`](05-desktop-application-design.md) (offline-first, MV* UI patterns, OS integration), and depends on [`06`](06-quality-attributes-tradeoffs.md) and [`07`](07-security-reliability-operations.md) for cross-cutting trade-offs and operations.

> **Mobile meta-principle:** *The device is borrowed, the battery is finite, and the OS can end your process without asking.* Design for interruption, degrade gracefully offline, ask for the minimum permission needed, and earn the user's trust in a store review before you earn it in the product.

---

## 1. What Makes Mobile Different

| Dimension | Desktop | Mobile |
|---|---|---|
| Process lifecycle | App controls its own lifetime | **OS controls it** — background limits, suspension, kill-to-reclaim-memory |
| Connectivity | Usually online | **Intermittent by default** — cellular, airplane mode, poor signal |
| Power | Plugged in or large battery | **Battery is a first-class constraint**; thermal throttling |
| Input | Mouse/keyboard, large screen | Touch/gesture, small screen, one-handed use, interruptions |
| Distribution & updates | Direct install, enterprise MSI/pkg | **App-store review gate**; staged rollout; users lag on updates |
| Permissions | Broad by default (user-owned OS) | **Explicit, runtime-granted, revocable** (camera, location, contacts…) |
| Identity/sensors | Optional peripherals | **Built-in**: GPS, biometrics, camera, accelerometer, push tokens |
| Context | Fixed workstation | Location-, motion-, and interruption-aware (calls, notifications) |
| Update reach | Enterprise push feasible | Auto-update common but **not guaranteed** — old versions persist for years |

#### Decision checklist
- What happens when the app is backgrounded mid-task, or the OS kills it to reclaim memory? Does the UI restore state correctly on relaunch?
- Which permissions are truly required, and can the feature degrade gracefully without them?
- What is the offline behavior — read-only, queued writes, or hard failure?
- Is store review timing (days, not minutes) compatible with your release cadence for this platform?

---

## 2. Mobile UI Architecture Patterns

The same MV* family from desktop ([05 §2](05-desktop-application-design.md#2-desktop-ui-architecture-patterns)) applies, with mobile-specific dominant idioms:

- **MVVM** — still common in traditional (imperative-view) Android (XML layouts) and legacy UIKit.
- **MVI (Model-View-Intent)** — a unidirectional variant popular in modern Android: a single immutable `State`, `Intent`s describing user actions, and a reducer producing the next state. Favors predictable state and easy testing under configuration changes (rotation, process death).
- **MVU / The Elm Architecture** — the natural fit for **declarative UI**: **SwiftUI** (`@State`/`@Observable`, unidirectional data flow) and **Jetpack Compose** (`State`, unidirectional flow, recomposition). See [05 §2.4](05-desktop-application-design.md#24-mvu--the-elm-architecture-unidirectional).
- **The Composable Architecture (TCA)** — a stricter MVU variant popular in Swift codebases: reducers, effects, and a single store, aimed at testability and structured concurrency.

#### Mobile-specific pressure on the pattern
Unlike a desktop process, a mobile screen's ViewModel/state holder must survive **configuration changes** (rotation, dark-mode switch, split-screen resize) and ideally **process death** (OS reclaiming memory while backgrounded) — state restoration is not optional polish, it's a correctness requirement. Android's `ViewModel` + `SavedStateHandle` and iOS's `@SceneStorage`/scene restoration exist specifically for this.

#### Decision checklist
- Does the state holder survive rotation and process death, or only foreground navigation?
- Is business logic separated from the platform view layer so it's unit-testable without simulators/emulators?

---

## 3. Native vs Cross-Platform: The Core Decision

Same fidelity-vs-reach spectrum as desktop ([05 §3](05-desktop-application-design.md#3-cross-platform-vs-native-the-core-decision)), with a mobile-specific middle option: frameworks that **share logic but keep native UI**.

```
Highest fidelity ◄──────────────────────────────────────────► Highest reach
 Native per-OS      Logic-shared, native UI      Full cross-platform UI
 (Swift/SwiftUI,     (Kotlin Multiplatform)        (Flutter, React Native)
  Kotlin/Compose)
```

| Approach | Fidelity | Effort for 2 platforms | Perf | Ecosystem reuse |
|---|---|---|---|---|
| Native per-OS (Swift+Kotlin) | Highest | High (2 codebases, 2 skillsets) | Highest | None |
| Kotlin Multiplatform (KMP) | High (native UI) | Medium (shared logic, native UI) | Native | Business logic only |
| Flutter | High, consistent (not native) | Low (1 codebase) | High (own renderer) | Full app |
| React Native | Moderate–high | Low (1 codebase) | Good (native components via Fabric) | Full app, JS ecosystem |
| .NET MAUI | Moderate | Low (1 codebase) | Good | Full app, .NET ecosystem |

**Market/technical context (verify currency — this space moves fast):** React Native retains the largest install base and ecosystem; Flutter (Dart, Skia/Impeller renderer) leads UI-consistency and animation-heavy apps with pixel-identical rendering across platforms; Kotlin Multiplatform has grown quickly by sharing only non-UI logic (networking, caching, business rules) while each platform keeps its native declarative UI (SwiftUI / Jetpack Compose) — attractive when native fidelity and platform-idiomatic UI matter more than a single UI codebase.

- **Go native or KMP** when platform fidelity, deep OS-API access, or best-in-class performance (games, AR/camera-heavy, pro audio) matters, or when the org already has separate iOS/Android teams that only want logic shared.
- **Go Flutter/React Native/MAUI** when time-to-market and a single team/codebase dominate, the UI can tolerate being "consistent" rather than "native," and the app's needs are mostly CRUD/content/forms.

#### Decision checklist
- Is pixel-perfect platform-native feel a requirement (regulated, flagship, or accessibility-sensitive app), or is "good enough and consistent" acceptable?
- Does the team have native mobile skills, web skills, or .NET/C# skills already?
- How much of the app is heavy native-API surface (camera pipelines, ARKit/ARCore, background audio, Bluetooth) vs standard CRUD/forms/content? Heavy native-API use erodes the cross-platform framework's advantage.
- What's the cost of two codebases vs the cost of framework lock-in and bridge/interop overhead?

---

## 4. Cross-Platform Frameworks Compared

*(Binary sizes, benchmarks, and market share evolve continuously — verify current specifics against the framework docs in [`09`](09-references.md) before a production decision.)*

### 4.1 Flutter (Dart, own rendering)
Compiles Dart to native ARM/x64 code; draws every pixel itself via **Skia/Impeller**, so UI is pixel-identical across iOS/Android (and reusable toward desktop/web, [05 §4.4](05-desktop-application-design.md#44-flutter-desktop-dart-own-rendering)). Strong hot-reload developer experience; leads in animation-heavy, custom-design UI. Con: does not use native widgets, so platform look-and-feel and some accessibility affordances must be deliberately matched.

### 4.2 React Native (JavaScript/TypeScript)
Renders through **native host components** (not a webview) via the **Fabric** renderer and a bridgeless JS↔native architecture (post-"New Architecture"). Familiar to web/React teams; huge ecosystem. Bridge/interop overhead has shrunk with Fabric but heavy native-module use still adds complexity.

### 4.3 Kotlin Multiplatform (KMP)
Shares **logic, not UI** — networking, persistence, business rules, validation compile to native code on both platforms, while UI stays fully native (**SwiftUI** on iOS, **Jetpack Compose** on Android). Preserves native fidelity and platform-idiomatic UX while eliminating logic duplication; requires maintaining two UI layers. Fastest-growing of the three as of the mid-2020s among teams that already have native iOS/Android engineers.

### 4.4 .NET MAUI
Single C#/XAML codebase compiling to native controls per platform (successor to Xamarin.Forms). Best fit for teams already invested in .NET; native-control rendering gives reasonable fidelity.

### 4.5 Comparison Summary
| Framework | Language | UI model | Native fidelity | Best for |
|---|---|---|---|---|
| Flutter | Dart | Own-drawn (Skia/Impeller) | Consistent, not native | Custom design systems, animation-heavy apps, multi-surface reuse |
| React Native | JS/TS | Native components via Fabric | High | Web/React teams, fast iteration, large module ecosystem |
| Kotlin Multiplatform | Kotlin | **Native UI** (shared logic only) | Native | Teams with native iOS+Android engineers wanting shared business logic |
| .NET MAUI | C#/XAML | Native controls | High | .NET shops, line-of-business apps |
| Native (Swift+Kotlin) | Swift, Kotlin | Native | Highest | Flagship apps, deep OS/hardware integration, games |

> Recall the **First Law** ([06 §3](06-quality-attributes-tradeoffs.md#3-the-inevitability-of-trade-offs)): every framework trades fidelity, footprint, team structure, and reach differently. Prototype the highest-risk native-API dependency before committing.

---

## 5. Platform Specifics: iOS vs Android

### 5.1 iOS
- **UI frameworks:** SwiftUI (declarative, MVU-flavored), UIKit (imperative, MVC/MVVM).
- **App lifecycle:** strict background-execution limits — background tasks must request time via `BGTaskScheduler`/background modes; the OS suspends and can terminate backgrounded apps at will.
- **Storage & secrets:** app sandbox container; **Keychain** for credentials/tokens (survives reinstall by default — decide deliberately whether that's wanted); `UserDefaults` for simple prefs only, never secrets.
- **Distribution:** **App Store review** (guideline-driven, days-scale turnaround), TestFlight for beta; enterprise/ad-hoc distribution for internal apps.
- **Design system:** Apple Human Interface Guidelines (HIG).
- **Permissions:** runtime-requested with a required usage-description string per capability (camera, location, health, etc.); App Tracking Transparency (ATT) gates cross-app tracking/IDFA.

### 5.2 Android
- **UI frameworks:** Jetpack Compose (declarative, MVU/MVI-flavored), legacy XML Views (MVP/MVVM).
- **App lifecycle:** `Activity`/`Fragment`/`ViewModel` lifecycle; **process death** while backgrounded is routine — `ViewModel` + `SavedStateHandle` restore UI state; long-running background work uses `WorkManager` (respects Doze/App Standby battery restrictions) rather than raw services; **foreground-service** types are restricted and must be justified to Play policy.
- **Storage & secrets:** scoped/app-private storage; **Android Keystore** for key material; `EncryptedSharedPreferences`/`EncryptedFile`/`SQLCipher` for data at rest; `Room` as the standard local-DB layer.
- **Distribution:** **Google Play** review (faster/more automated than iOS, but with its own policy risks — e.g., permission justification, target-SDK deadlines), staged rollout **tracks** (internal → closed → open → production %). Sideloading/APK distribution possible outside Play.
- **Design system:** Material Design (Material 3).
- **Permissions:** runtime-requested, **individually revocable** by the user at any time (must re-check, not just re-request, before every sensitive operation); scoped storage restricts broad filesystem access by default.

### 5.3 Per-Platform Conventions Cheat Sheet
| Concern | iOS | Android |
|---|---|---|
| Declarative UI | SwiftUI | Jetpack Compose |
| Local DB | Core Data / SQLite / GRDB | Room (SQLite) |
| Secure storage | Keychain | Keystore + EncryptedSharedPreferences |
| Background work | `BGTaskScheduler`, background modes | `WorkManager` |
| Push | APNs | Firebase Cloud Messaging (FCM) |
| Store review | Human + automated, days-scale | Mostly automated, hours–days |
| Staged rollout | TestFlight, phased release | Play Console rollout tracks |
| Design system | Human Interface Guidelines | Material Design |
| Signing | Automatic/manual provisioning profiles | App Signing by Google Play (upload key + app signing key) |

---

## 6. Offline-First & Local Data / Sync

Mobile connectivity is intermittent by default, not exceptional — treat offline as the normal case, not an edge case. The local-storage options, sync-strategy ladder (LWW → version vectors → CRDTs → user-prompted), and "make sync status visible" guidance from desktop apply directly: see [05 §7](05-desktop-application-design.md#7-offline-first--local-data).

Mobile-specific additions:
- **Local DB choice:** Room (Android/KMP), Core Data or a SQLite wrapper such as GRDB (iOS), or a cross-platform embedded DB (SQLite, Realm, ObjectBox) when sharing a data layer across a Flutter/React Native/KMP codebase.
- **Sync engines:** purpose-built sync layers (e.g., CRDT-based or operational-transform sync services) reduce hand-rolled conflict logic for collaborative or multi-device apps.
- **Queued writes:** mutations made offline should enqueue (an outbox, [02 §5.9](02-architecture-patterns.md#59-transactional-outbox)) and replay on reconnect with idempotency ([02 §7.7](02-architecture-patterns.md#77-idempotency)) so a flaky network never silently drops user input.
- **Storage budget:** mobile devices have far less local storage headroom than desktops/servers — cap and evict local caches deliberately; never let an offline cache grow unbounded.

---

## 7. State, Navigation, Deep Links & Push

- **Navigation:** platform-native stack/tab navigation (`NavigationStack` on iOS, `Navigation` on Compose) or a cross-platform router; preserve back-stack state across process death where the platform expects it.
- **Deep links / app links / universal links:** a URL or platform-verified link (Android App Links, iOS Universal Links) that opens the app directly to a specific screen/state — required for marketing, notification taps, and OAuth redirect flows. Verify link ownership (`assetlinks.json` / `apple-app-site-association`) so links can't be hijacked by another app.
- **Push notifications:** APNs (iOS) / FCM (Android), typically via a unified provider. Treat the push token as sensitive (bind it to a user session server-side, rotate/invalidate on logout); request notification permission with a clear rationale rather than at first launch (a common store-rejection and opt-out driver).
- **Background work:** schedule via the OS-provided scheduler (`WorkManager`, `BGTaskScheduler`) rather than fighting the OS with wake-locks or polling loops — the OS *will* win, and aggressive background behavior triggers battery warnings, OS throttling, and store scrutiny.

---

## 8. Mobile Security

#### Threat model
The device can be lost, stolen, jailbroken/rooted, or running alongside malicious apps; the app binary can be decompiled/reverse-engineered; network traffic may cross untrusted Wi-Fi. Unlike a desktop app run by a single trusted user, a mobile app's install base spans a huge range of device-security postures, so **defense in depth on-device** matters more.

### 8.1 OWASP MASVS & MASTG
The authoritative standard for mobile app security is the **OWASP Mobile Application Security Verification Standard (MASVS)**, tested via the companion **Mobile Application Security Testing Guide (MASTG)**:

- **MASVS-L1** — baseline security every mobile app should meet (secure storage, transport security, basic platform interaction hygiene).
- **MASVS-L2** — defense-in-depth for apps handling sensitive data (finance, health, enterprise): stronger crypto/key management, additional authentication assurance.
- **MASVS-R (Resilience)** — anti-tampering and reverse-engineering resistance (obfuscation, anti-debugging, root/jailbreak detection) for apps facing targeted attackers.

Use MASVS to decide **what** to verify and MASTG for **how** to test it (step-by-step, tool-backed test cases per platform). Treat MASVS-L1 as close to non-negotiable; reserve L2/R for apps where the risk (financial loss, health data, high-value IP) justifies the added friction and cost.

### 8.2 Core Mobile Security Practices
| Practice | Guidance |
|---|---|
| Secure storage | iOS **Keychain**; Android **Keystore** + `EncryptedSharedPreferences`/`EncryptedFile`; never plaintext secrets in `UserDefaults`/`SharedPreferences` or app files |
| Transport security | TLS everywhere; consider **certificate pinning** for high-value apps (weigh against operational risk of pin rotation failures) |
| No secrets in the binary | API keys, signing secrets, and backend credentials do not belong in the client — proxy sensitive calls through a backend the client authenticates to |
| Authentication | short-lived tokens; **biometric auth** (Face ID/Touch ID, Android BiometricPrompt) as a local unlock gate in front of a securely stored credential, not as the sole server-side authentication factor |
| App signing | latest signing scheme (Android App Signing v2/v3; iOS provisioning + Apple signing); protect the upload/signing key |
| Jailbreak/root awareness | detect and respond proportionately (warn, restrict sensitive features) rather than silently trusting a compromised runtime for high-risk apps |
| Input/IPC validation | validate all inputs from intents/URL schemes/deep links/clipboard — they are attacker-reachable surfaces |
| WebView hygiene | if embedding a WebView, disable unnecessary JS bridges, validate loaded origins, treat it as a hostile boundary (same reframe as desktop web-tech shells, [05 §9](05-desktop-application-design.md#9-desktop-security)) |

### 8.3 Common Mistakes
- Shipping API keys or backend admin credentials inside the app bundle (trivially extracted by decompilation).
- Trusting client-side permission/role checks with no server-side enforcement — mirrors the desktop/web mistake of trusting the client ([07 §3](07-security-reliability-operations.md#3-authentication-authorization-and-identity)).
- Relying on biometric unlock alone with no server-side session/token validation behind it.
- Skipping certificate/TLS validation "for convenience" in a debug build that ships to production.
- No MASVS baseline check before release for apps handling payment, health, or authentication data.

---

## 9. Performance & Resource Use

Mobile devices share desktop's "the app shares finite resources" constraint ([05 §10](05-desktop-application-design.md#10-performance--resource-use)) but with sharper limits and OS enforcement:

| Area | Techniques |
|---|---|
| Cold start | Lazy-init non-critical singletons/SDKs; defer heavy work past first frame; measure cold vs warm start explicitly |
| Jank / frame budget | Keep work off the main/UI thread; target the platform's frame budget (commonly ~16 ms for 60 Hz, tighter at 120 Hz) — see the frame-budget discipline in [10 §15](10-game-architecture.md#15-frame-budget--profiling-discipline), which applies equally to UI rendering |
| Memory | Mobile memory limits are tight and OS-enforced — exceeding them gets the process killed; profile allocations, bound image/caches |
| Battery/network | Batch network calls; avoid polling (use push instead); respect Doze/App Standby and iOS background-refresh budgets; avoid unnecessary GPS/sensor polling |
| App size | App-store download-size limits and cellular-download thresholds affect install conversion; use platform app-size-reduction tooling (App Bundles/App Thinning) |
| Thermal | Sustained heavy CPU/GPU work triggers OS thermal throttling — design for sustained, not just peak, load |

Profile on real, low-to-mid-tier devices, not just flagship simulators/emulators — the performance gap between a high-end and budget device is far larger on mobile than on desktop.

---

## 10. Accessibility

Mobile accessibility builds on the same POUR framework as web ([04 §5](04-web-application-design.md#5-accessibility-a11y)) with platform-native tooling:

- **iOS:** VoiceOver (screen reader), Dynamic Type (user-scalable text), Switch Control, Reduce Motion.
- **Android:** TalkBack (screen reader), font scaling, `contentDescription` for non-text elements.
- **Touch targets:** minimum recommended touch-target size (commonly cited around 44×44pt/48×48dp) to accommodate motor-impairment and general usability.
- **Contrast, focus order, and semantic labeling** apply exactly as in web WCAG guidance — see [04 §5](04-web-application-design.md#5-accessibility-a11y) and the standards crosswalk ([`13`](13-standards-crosswalk.md)) for WCAG's applicability to native apps via platform accessibility APIs.

---

## 11. Packaging, Signing, Store Submission & Release

### 11.1 Build Artifacts
- **iOS:** `.ipa`, built and signed via Xcode/App Store Connect tooling; provisioning profiles bind an app to devices/entitlements during development.
- **Android:** **Android App Bundle (`.aab`)** is the modern Play-required format (Play generates optimized per-device APKs); raw `.apk` remains for sideloading/enterprise distribution.

### 11.2 Signing
- **iOS:** Apple-managed signing identities + provisioning profiles; Apple ultimately re-signs/wraps App Store distributions.
- **Android:** **App Signing by Google Play** separates the developer's **upload key** from Google's managed **app signing key** — protects against upload-key compromise while Google retains the signing key for the app's lifetime.
- The signing/upload key is a "crown jewel" exactly as on desktop ([05 §11.2](05-desktop-application-design.md#112-code-signing--notarization)) — store it in a secrets manager/HSM or the store's managed-signing service, never in the repo.

### 11.3 Store Submission & Staged Rollout
- **Beta/internal testing:** TestFlight (iOS), Play Console internal/closed/open testing tracks (Android).
- **Review:** budget days (iOS) to hours-or-days (Android) of review lead time into release planning; a rejected build blocks the whole release train, so keep a rollback-capable release process.
- **Staged rollout:** ship to a percentage of users first (Play rollout percentage; iOS phased release) and watch crash/ANR rates and key metrics before ramping to 100% — mirrors canary deployment ([04 §14](04-web-application-design.md#14-deployment-strategies)) but on a days-long timescale because you cannot instantly roll back a store release the way you can a server deploy.
- **Over-the-air (OTA) update layers** (e.g., CodePush-style JS/asset updates for React Native, or Expo/EAS updates) can patch non-native code without a full store review cycle — useful for fast content/bugfix iteration, but **cannot** touch native code, must still respect store policies (some stores restrict what OTA updates may change), and should never be used to bypass review for functionality changes.
- **Version lag:** unlike a web app, you cannot force every user onto the latest version — design server-side APIs to tolerate several concurrent client versions, and set an explicit minimum-supported-version policy with a forced-upgrade prompt for security-critical fixes.

---

## 12. Mobile-Specific Quality Attributes & Anti-Patterns

Re-weighting the quality-attribute catalog ([06 §2](06-quality-attributes-tradeoffs.md#2-the-quality-attributes-catalog); see also the ISO/IEC 25010 mapping in [`13`](13-standards-crosswalk.md)) for mobile:

| Attribute | Mobile emphasis |
|---|---|
| Battery/thermal efficiency | **Elevated** — a battery-draining app gets uninstalled and store-flagged |
| Offline resilience | **Elevated** — intermittent connectivity is the default state |
| Resumability / state restoration | **Elevated** — process death and backgrounding are routine, not exceptional |
| Store-review compliance | **New axis** — a policy violation blocks distribution entirely, unlike a web deploy |
| Update reach / version fragmentation | **Elevated** — old client versions persist for years; server APIs must tolerate this |
| Security (device-loss threat model) | **Reframed** — the device itself can be a hostile environment (lost/rooted) |
| Accessibility | **Elevated** — platform screen readers are heavily used; app-store accessibility scrutiny is increasing |
| Portability | The native/cross-platform axis ([§3](#3-native-vs-cross-platform-the-core-decision)) |

### Anti-Patterns
| Anti-pattern | Why harmful | Avoid by |
|---|---|---|
| Blocking the main/UI thread | Frozen UI, ANR (Application Not Responding) dialogs, store-visible crash-rate penalties | Async/background work off the main thread (§7, §9) |
| Ignoring process-death state restoration | User loses in-progress work on backgrounding | `ViewModel`+`SavedStateHandle` / scene restoration (§2) |
| Polling instead of push | Battery drain, OS throttling, worse latency | Push notifications + OS-managed background scheduling (§7, §9) |
| Secrets or API keys in the client binary | Trivially extracted by decompilation | Proxy sensitive calls through an authenticated backend (§8.2) |
| Requesting broad permissions upfront | Store rejection risk; user distrust; opt-out | Request permissions in-context, minimally, with rationale (§1, §7) |
| No offline handling (hard failure with no network) | App feels broken on flaky connections, the mobile norm | Offline-first local data + outbox sync (§6) |
| Assuming instant update reach | Server breaks for the long tail of old clients | Version-tolerant APIs, minimum-supported-version policy (§11.3) |
| Trusting client-side checks alone | Bypassable by a rooted/jailbroken device or a patched APK/IPA | Server-side enforcement of every security-relevant decision (§8) |
| Aggressive background wake-locks | OS battery warnings, app store scrutiny, user uninstalls | `WorkManager`/`BGTaskScheduler`, respect Doze/thermal states (§7, §9) |
| Skipping low-end device testing | Performance regressions invisible on flagship devices/simulators | Profile on real mid/low-tier hardware (§9) |

---

## Key Cross-References
- **Architecture & design:** [`01`](01-architecture-principles.md), [`02`](02-architecture-patterns.md), [`03`](03-software-design-principles.md).
- **Shares UI-pattern and offline/sync foundations with:** [`05` — Desktop Application Design](05-desktop-application-design.md).
- **Web-adjacent concerns (accessibility, API design, OAuth/PKCE):** [`04`](04-web-application-design.md).
- **Quality attributes & trade-offs:** [`06`](06-quality-attributes-tradeoffs.md).
- **Security, supply chain, delivery:** [`07`](07-security-reliability-operations.md).
- **Standards crosswalk (MASVS, ISO/IEC 25010, WCAG, EN 301 549):** [`13`](13-standards-crosswalk.md). **Checklists:** [`08`](08-checklists-and-templates.md).

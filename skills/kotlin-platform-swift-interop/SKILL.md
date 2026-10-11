---
name: kotlin-platform-swift-interop
description: Use when a task crosses the boundary between a Kotlin Multiplatform (KMP) shared module and native Swift/SwiftUI code on iOS — exposing the shared module's framework to Swift, consuming suspend functions/Flow/sealed classes/errors from SwiftUI, threading rules across the Kotlin/Native-Swift boundary, and deciding whether a mobile task needs the Kotlin KMP skills, the swiftui-expert-skill, or both. Also use to route any cross-platform (Android + iOS) feature, review, or bugfix task to the right combination of skills in this catalog.
allowed-tools: Read, Grep, Glob
license: Apache-2.0
metadata:
  note: New skill written to bridge kotlin-kmp-claude-agent-skills (Apache-2.0, Mariano Miani) and swiftui-expert-skill (MIT, Antoine van der Lee & Omar Elsayed); does not reproduce either's text. Distributed under this catalog's Apache-2.0 license.
  version: "1.0.0"
  sources: "https://kotlinlang.org/, https://commonmain.dev/"
---

# Kotlin Multiplatform ↔ Swift/SwiftUI Bridge

Use this skill first for any task that touches **both** a Kotlin Multiplatform shared module and a native iOS UI written in SwiftUI. It does two things:

1. **Routes** the task to the right skill(s) in this catalog (the 15 `kotlin-*` KMP skills and `swiftui-expert-skill`), so you load only what the task needs.
2. **Owns the gap** between them: the Swift-facing API surface of the Kotlin/Native framework — the one layer neither source skill set covers, because the KMP skills stop at `iosMain` and the SwiftUI skill assumes a pure-Swift codebase.

Do not duplicate the content of the routed-to skills here. Read them for their domain; come back to this skill only for the cross-boundary concerns in the sections below.

## Which architecture are you in?

KMP mobile apps follow one of two shapes. Identify which one before routing — the mix of skills you need differs sharply.

**A. Shared UI (Compose Multiplatform on both platforms).**
Kotlin owns the UI tree on Android *and* iOS via Compose Multiplatform; there is little or no SwiftUI beyond a thin native host (`ComposeUIViewController` wrapped by a tiny SwiftUI/UIKit entry point) and maybe a handful of native-only screens (widgets, share extensions, native-only settings pages).
→ Route almost everything to the `kotlin-ui-*`, `kotlin-navigation-*`, and `kotlin-project-*` skills. Reach for `swiftui-expert-skill` only for the native-only screens, and for this skill only to review the thin SwiftUI host shell itself.

**B. Shared logic, native UI per platform.**
The Kotlin shared module (`commonMain`/`iosMain`) holds business logic, state holders, and data; Android renders it with Jetpack Compose, iOS renders it with SwiftUI consuming the compiled Kotlin/Native framework directly.
→ Both skill sets apply in full, and this skill's interop guidance is load-bearing on every feature that crosses the boundary.

Most of this file assumes shape B. In shape A, skip to [Thin native host review](#thin-native-host-review-shape-a) at the end.

## Routing table

| Task touches... | Load |
|---|---|
| `commonMain` business logic, domain models, use cases | [`kotlin-project-architecture-review`](../kotlin-project-architecture-review/SKILL.md), [`kotlin-project-feature-implementation`](../kotlin-project-feature-implementation/SKILL.md) |
| Repositories, data sources, caching/sync | [`kotlin-data-kmp-data-layer`](../kotlin-data-kmp-data-layer/SKILL.md) |
| State holders consumed by *either* platform UI (ViewModel, shared presenter, MVI, `StateFlow`) | [`kotlin-project-state-management`](../kotlin-project-state-management/SKILL.md) |
| `expect`/`actual`, source-set placement, `commonMain`/`androidMain`/`iosMain` boundaries | [`kotlin-platform-kmp-bridges`](../kotlin-platform-kmp-bridges/SKILL.md) |
| The compiled framework's Swift-facing API, consuming it from SwiftUI, Flow/suspend/error mapping | **this skill**, [reference/swift-interop.md](reference/swift-interop.md) |
| Shared Compose UI (Android + iOS via Compose Multiplatform) | [`kotlin-ui-compose-multiplatform`](../kotlin-ui-compose-multiplatform/SKILL.md), [`kotlin-ui-adaptive-resources`](../kotlin-ui-adaptive-resources/SKILL.md), [`kotlin-navigation-compose-multiplatform`](../kotlin-navigation-compose-multiplatform/SKILL.md) |
| Native SwiftUI views, state (`@State`/`@Observable`), animations, lists, performance, Instruments traces | [`swiftui-expert-skill`](../swiftui-expert-skill/SKILL.md) |
| Android deep links / App Links | [`kotlin-platform-app-links-and-deep-links`](../kotlin-platform-app-links-and-deep-links/SKILL.md) |
| Gradle structure, iOS framework packaging task, version catalogs | [`kotlin-build-kmp-gradle-governance`](../kotlin-build-kmp-gradle-governance/SKILL.md), [reference/packaging.md](reference/packaging.md) |
| Shared/platform test strategy, fakes for SwiftUI previews | [`kotlin-testing-kmp`](../kotlin-testing-kmp/SKILL.md) |
| Module boundaries | [`kotlin-project-modularization`](../kotlin-project-modularization/SKILL.md) |
| Bugfix spanning shared + native code | [`kotlin-project-bugfix`](../kotlin-project-bugfix/SKILL.md) |
| Refactor/migration spanning shared + native code | [`kotlin-kmp-refactor-safety`](../kotlin-kmp-refactor-safety/SKILL.md) |
| Already-implemented KMP/Compose code review | [`kotlin-kmp-code-review`](../kotlin-kmp-code-review/SKILL.md) |

A single feature almost always needs 2–4 skills in sequence: typically a `kotlin-project-*` or `kotlin-data-*` skill for the shared side, this skill for the boundary, and `swiftui-expert-skill` (or the `kotlin-ui-*` skills) for the UI side.

## Cross-platform feature workflow

1. **Design the shared contract first.** Decide what the Kotlin shared module exposes — not how. Use `kotlin-project-feature-implementation` and `kotlin-platform-kmp-bridges` for this; the contract should be shaped for the consumer, not for Kotlin's convenience (see [Shared contract quality](../kotlin-platform-kmp-bridges/SKILL.md)).
2. **Decide the Swift-facing shape of that contract** before writing either side — suspend fn vs. callback vs. Swift `async`, whether a `Flow` needs wrapping, whether errors must cross via `@Throws`. See [reference/swift-interop.md](reference/swift-interop.md). Getting this wrong means rewriting both the Kotlin API and the SwiftUI view later.
3. **Implement the Kotlin side** against `kotlin-project-feature-implementation` / `kotlin-data-kmp-data-layer` rules.
4. **Implement the SwiftUI side** against `swiftui-expert-skill`'s "Implement new SwiftUI feature" workflow, treating the injected Kotlin state holder as the "owned vs injected state" boundary it describes — the Kotlin side owns it, SwiftUI observes it (see [reference/swift-interop.md § Observing shared state from SwiftUI](reference/swift-interop.md#observing-shared-state-from-swiftui)).
5. **Test both sides independently.** Shared logic gets `kotlin-testing-kmp` coverage with fakes; SwiftUI previews use the same fakes exposed through the framework, never the live shared module (self-contained preview data is a hard rule in `swiftui-expert-skill`'s correctness checklist).
6. **Review each side with its own skill**, then re-check the boundary itself: did the Swift-facing shape from step 2 survive intact, or did either side quietly route around it?

## Thin native host review (shape A)

When Compose Multiplatform owns the UI, the only SwiftUI/UIKit code is the app entry point wrapping `ComposeUIViewController` (or a `UIHostingController`/`UIViewControllerRepresentable` bridge). Review it narrowly:

- it contains no business logic, no state, no navigation decisions — those stay in the shared Compose UI
- `@main` App struct and scene setup are minimal: construct the shared module's root dependencies (or DI container) and hand control to Compose
- lifecycle events that Compose Multiplatform needs from iOS (e.g., safe-area/keyboard notifications, scene-phase changes) are forwarded, not reinterpreted
- if any native-only SwiftUI screens exist alongside Compose screens, treat navigation *between* a Compose screen and a native screen as a platform bridge: review it with this skill's interop guidance, not as ordinary SwiftUI navigation

## Correctness checklist (interop-specific)

- [ ] Every shared-module API consumed by Swift that can throw declares `@Throws(ExceptionClass::class)`; nothing relies on an undeclared Kotlin exception reaching Swift (it terminates the process)
- [ ] No raw `Flow` is exposed directly across the framework boundary without a documented consumption strategy (native `async`/Combine/callback wrapper) — see [reference/swift-interop.md](reference/swift-interop.md)
- [ ] Suspend functions intended for direct Swift `async`/`await` use are verified against the Kotlin/Swift concurrency interop support in the project's Kotlin version, with a completion-handler fallback path understood
- [ ] Collections crossing the boundary are cast to their `NS*` counterparts at the call site when touched in a hot path, instead of relying on repeated bridging conversions
- [ ] Kotlin `object`/companion singletons consumed from Swift are accessed via `.shared`/`.companion`, not re-instantiated on the Swift side
- [ ] SwiftUI views hold the Kotlin state holder as injected state (per `swiftui-expert-skill`'s state rules), never re-create or duplicate it locally
- [ ] SwiftUI previews inject fake/mock implementations of the shared contract — never the live shared module — per both skills' testability rules
- [ ] The Gradle multiplatform iOS target and the Xcode build step that consumes it are reviewed together when either changes (see [reference/packaging.md](reference/packaging.md))

## References

- Kotlin Multiplatform docs: https://kotlinlang.org/docs/multiplatform/multiplatform-discover-project.html
- Expected and actual declarations: https://kotlinlang.org/docs/multiplatform/multiplatform-expect-actual.html
- Use platform-specific APIs: https://kotlinlang.org/docs/multiplatform/multiplatform-connect-to-apis.html
- Kotlin/Native interop with Swift/Objective-C: https://kotlinlang.org/docs/native-objc-interop.html
- Kotlin/Native and ARC integration: https://kotlinlang.org/docs/native-arc-integration.html
- commonMain.dev — curated Kotlin Multiplatform newsletter and guide index: https://commonmain.dev/

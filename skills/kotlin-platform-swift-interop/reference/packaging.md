# Packaging the shared module for Xcode/SwiftUI consumption

This is the build-plumbing half of the boundary: `kotlin-build-kmp-gradle-governance` reviews Gradle structure in general; this section is specifically about the step where that Gradle output becomes something Xcode and a SwiftUI target can link against.

## What crosses the boundary

The Kotlin Multiplatform Gradle plugin compiles the iOS targets (`iosArm64`, `iosSimulatorArm64`, and `iosX64` if the project still declares it) into a Kotlin/Native framework. For distribution as a single artifact covering device and simulator slices, that framework is assembled into an **XCFramework**. Either way, Xcode links the framework, and its generated Objective-C header is what the mapping rules in [swift-interop.md](swift-interop.md) describe.

## The five official integration methods

The current Kotlin Multiplatform iOS-integration docs name five distinct setups. Identify which one a project actually uses before diagnosing a "my Kotlin change isn't showing up in the iOS app" report — each has a different staleness failure mode.

| Method | What happens | When it's the right fit |
|---|---|---|
| **Direct integration** | An Xcode "Run Script" build phase invokes the Gradle `embedAndSignAppleFrameworkForXcode` task on every Xcode build, so the Kotlin framework is rebuilt, embedded, and code-signed as part of the iOS build. (This task replaced the older `packForXcode` task starting with Kotlin 1.5.20 — if a project's script still calls `packForXcode`, that is dated setup worth flagging.) | Single-repo app, no CocoaPods in the KMP project. This is the default the Kotlin Multiplatform IDE plugin sets up. |
| **SwiftPM local integration** | The iOS project depends on the Kotlin framework as a local Swift package. | Monorepo where the iOS project already uses Swift Package Manager and has no hard CocoaPods dependency. |
| **CocoaPods local integration** | The iOS project depends on the Kotlin framework through a local podspec via the Kotlin CocoaPods Gradle plugin. | Monorepo where the iOS project already uses CocoaPods, or the KMP module itself needs to consume CocoaPods dependencies. |
| **SwiftPM remote integration** | The Kotlin module is built as an XCFramework and distributed as a remote Swift package. | Shared code is versioned and released separately from the apps that consume it. |
| **CocoaPods remote integration** | The Kotlin module is built as an XCFramework via the Kotlin CocoaPods Gradle plugin and distributed as a third-party CocoaPods dependency. | Same separation as above, for teams standardized on CocoaPods distribution instead of SwiftPM. |

The two failure modes to tell apart:

- **Direct integration, stale build**: the Gradle task silently didn't re-run (wrong working directory or `PATH` in the Xcode build phase, often only reproducing on CI, not on a developer machine with a working local toolchain) — the fix is in the Xcode build phase script, not the Kotlin code.
- **Remote/local SwiftPM or CocoaPods, stale framework**: the consumed framework version is pinned to (or floating on) a build that predates the latest `commonMain`/`iosMain` change — the fix is republishing and bumping the dependency, not touching the Xcode project.

## Review checklist

- [ ] The integration method actually in use is identified (one of the five above) before any "framework isn't updating" diagnosis proceeds
- [ ] If using direct integration, the build phase calls `embedAndSignAppleFrameworkForXcode`, not the deprecated `packForXcode`
- [ ] The iOS target list in the Gradle KMP configuration matches what's actually shipped (device + simulator architectures) — no targets declared out of habit that nothing in the project actually builds for
- [ ] Debug and release framework configurations are both exercised — a framework built only in debug can hide release-mode Kotlin/Native optimizer behavior until a release build
- [ ] If using direct integration, the build phase's working directory and `PATH` assumptions are verified on CI specifically, not only on a developer machine
- [ ] If distributing via SwiftPM or CocoaPods (local or remote), the consumed framework version is pinned deliberately — not floating in a way that lets Swift code silently run against a shared module that no longer matches the Kotlin repo's current `commonMain`/`iosMain` source
- [ ] The generated Objective-C header is treated as build output, not hand-edited or relied upon for anything beyond what `@ObjCName`/`@Throws`/KDoc annotations in the Kotlin source already control

## References

- Kotlin Multiplatform: iOS integration methods (overview): https://kotlinlang.org/docs/multiplatform/multiplatform-ios-integration-overview.html
- Kotlin Multiplatform: Direct integration: https://kotlinlang.org/docs/multiplatform/multiplatform-direct-integration.html
- Kotlin Multiplatform: Import local Swift packages (SwiftPM local integration): https://kotlinlang.org/docs/multiplatform/multiplatform-spm-import.html
- Kotlin Multiplatform: Export to a Swift package (SwiftPM remote integration): https://kotlinlang.org/docs/multiplatform/multiplatform-spm-export.html
- Kotlin Multiplatform: CocoaPods overview: https://kotlinlang.org/docs/native-cocoapods.html
- Kotlin Multiplatform: Build final native binaries (XCFrameworks): https://kotlinlang.org/docs/multiplatform/multiplatform-build-native-binaries.html
- Kotlin Multiplatform DSL reference: https://www.jetbrains.com/help/kotlin-multiplatform-dev/multiplatform-dsl-reference.html

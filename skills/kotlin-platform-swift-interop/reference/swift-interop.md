# Swift-facing API surface of the Kotlin/Native framework

This is the layer between `kotlin-platform-kmp-bridges` (which owns `expect`/`actual` and source-set placement *inside* the Kotlin module) and `swiftui-expert-skill` (which owns the SwiftUI view code). It covers what the Kotlin shared module looks like once it crosses into Swift, and how a SwiftUI view should consume it.

Kotlin/Native does not talk to Swift directly — it goes through an **Objective-C bridging header** that Swift then imports. Every mapping rule below is a consequence of that: a construct that Objective-C cannot represent (raw generics, Flow, multiple return values, operator overloading on arbitrary types) does not cross cleanly and needs a deliberate shape decision.

## Type and construct mapping

| Kotlin | Crosses as | Notes |
|---|---|---|
| `fun f(): T` | `func f() -> T` | Direct for ordinary types |
| `suspend fun f(): T` | Obj-C: method with `completionHandler:`. Swift 5.5+: can also be called with native `async`/`await` (experimental; version/toolchain dependent — verify against the project's Kotlin and Xcode versions before relying on it) | Prefer the completion-handler form as the documented baseline; treat native `async` call-through as an optimization to verify, not assume |
| `@Throws(E::class) fun f()` | Obj-C `NSError**` out-param / Swift `throws` | **Only exceptions listed in `@Throws` propagate as catchable errors.** Any other Kotlin exception reaching the Swift boundary terminates the process. Every throwing shared-module API needs an explicit, reviewed `@Throws` list — do not let it default to "whatever the implementation currently throws" |
| `Flow<T>` | No native representation | See [Exposing Flow to SwiftUI](#exposing-flow-to-swiftui) below — this is the one construct that needs a conscious wrapping decision on every API that uses it |
| `List` / `Map` / `Set` | `Array` / `Dictionary` / `Set` (Swift), bridging through `NSArray`/`NSDictionary`/`NSSet` | Bridging has a real cost on hot paths. If a Swift call site reads the same collection repeatedly, cast once: `let nsMap = map as NSDictionary` and work with the `NS*` type rather than re-bridging on every access |
| `enum class` | Swift `class` with one static property per case (not a Swift `enum`) | `Colors.RED` becomes `Colors.red` (lowercased, following Obj-C-import casing). Do not assume Swift `switch` exhaustiveness checking behaves exactly like a native Swift enum — verify against the actual generated header |
| `sealed class` / `sealed interface` | Imported the same way as a regular class hierarchy (subclasses as subclasses), not as a Swift `enum` with associated values | There is no automatic exhaustive `switch` over a Kotlin sealed hierarchy the way there is over a Swift enum. Model the Swift-side consumption explicitly — typically an `if`/`else if` chain with a final `fatalError`/default case, or a deliberate mapping function Kotlin-side that projects the sealed hierarchy into a Swift-friendly shape before export |
| Function types / lambdas | Obj-C block pointers → Swift closures | Primitive parameters in the lambda signature are **boxed** (`(Int) -> Unit` becomes `(KotlinInt) -> KotlinUnit` on the Swift side) — not free |
| `object` (singleton) | `TheObject.shared` | `companion object` members are reached via `TheClass.companion` |
| Inline classes | **Not supported across the boundary** | Do not expose a Kotlin `value class`/inline class directly in a public shared-module API; unwrap it to its underlying type first |

## Exposing Flow to SwiftUI

`Flow` has no Swift representation — it is a Kotlin coroutines construct with no Objective-C equivalent. Every shared-module API that returns a `Flow<T>` needs one deliberate decision, made at design time (step 2 of the cross-platform feature workflow in `SKILL.md`), not discovered while writing the SwiftUI view:

- **Callback/polling wrapper**: expose a plain function that takes a Swift closure and returns a "cancellation handle" object, internally doing `flow.onEach { callback(it) }.launchIn(scope)`. Simple, no extra dependency, but hand-rolled cancellation and backpressure.
- **Expose the latest value only**: if the SwiftUI view only ever needs "current state," expose a plain property/suspend getter backed by `StateFlow.value` instead of the stream, and have the view poll or re-fetch on a lifecycle event rather than subscribing to a stream at all.
- **Third-party bridging library** (e.g. community tools that generate `AsyncSequence`/Combine wrappers for suspend functions and Flows): evaluate current options against the project's Kotlin version before adopting one; this is fast-moving tooling, not something to treat as a permanent official answer. Whatever is chosen, keep the choice confined to the framework boundary — shared `commonMain` code should never be written *around* a specific Swift bridging tool.

Whichever shape is chosen, document it next to the shared API, because it determines how the SwiftUI view's `@Observable`/`@State` wiring in `swiftui-expert-skill`'s [`state-management.md`](../../swiftui-expert-skill/references/state-management.md) needs to look.

## Observing shared state from SwiftUI

The Kotlin shared module is the state owner; SwiftUI only observes. Concretely:

- Wrap the Kotlin state holder (ViewModel / shared presenter — see `kotlin-project-state-management`) in a thin Swift type that SwiftUI can observe idiomatically: on iOS 17+, an `@Observable` wrapper class that subscribes to the shared `StateFlow` (via whichever Flow-exposure shape was chosen above) and republishes value changes as its own stored properties.
- The SwiftUI view then holds that wrapper the same way `swiftui-expert-skill` describes for any `@Observable`-backed dependency: `@State` if the view creates/owns the wrapper instance, injected via `@Environment`/initializer if it doesn't. The *Kotlin* state is still the single source of truth; the Swift wrapper is a projection, not a second owner — do not let the wrapper accumulate its own independent `@State` that can drift from the Kotlin side.
- Events/effects flow the opposite direction as plain function calls into the shared module (per the unidirectional-data-flow default in the KMP skills) — not as a second notification channel back through the wrapper.

## Threading

- Kotlin coroutines dispatchers are a Kotlin-side concern; nothing about crossing into Swift changes that a `Flow`/suspend function should already be main-safe per `kotlin-data-kmp-data-layer`'s main-safe-operations guidance — the shared module should not require its Swift caller to know which thread a callback arrives on.
- If a completion handler's documented threading contract is unclear (the official interop docs do not enforce a specific thread for completion-handler delivery), the shared module should state and guarantee the thread explicitly — most commonly by redispatching onto the main thread itself before invoking the Swift callback — rather than leaving SwiftUI to defensively `DispatchQueue.main.async` every call.
- ARC and Kotlin/Native's memory model interact at the boundary (reference cycles between Swift closures held by Kotlin and vice versa). When a SwiftUI view passes a closure into a long-lived Kotlin object (e.g., a callback registered for the lifetime of a shared ViewModel), check for retain cycles explicitly — this is a distinct concern from normal Swift ARC and is not covered by `swiftui-expert-skill`.

## Errors

- Only exceptions declared in `@Throws` on the Kotlin function become catchable Swift errors. Treat an undeclared exception crossing the boundary as equivalent to a Swift `fatalError` — it crashes.
- Design the `@Throws` list around the Swift caller's recovery options (what can the SwiftUI view actually do differently per error case?), not around "every exception type the implementation happens to throw today." An overly broad or stale `@Throws` list is a shared-contract quality problem — flag it the same way `kotlin-platform-kmp-bridges` flags vendor-shaped shared contracts.

## Verifying this reference against current Kotlin releases

Kotlin/Native's Swift interop story moves (completion-handler-vs-native-async support, and newer "Swift export" tooling that may change how much of this mapping table is hand-written vs. generated). Before relying on a specific claim above for a production decision, check it against the current docs:

- https://kotlinlang.org/docs/native-objc-interop.html
- https://kotlinlang.org/docs/multiplatform/multiplatform-connect-to-apis.html
- https://kotlinlang.org/docs/releases.html (stability notes — Kotlin/Swift interop features have shipped as experimental/beta before stabilizing)

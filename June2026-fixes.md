# Windule Android App - Crashlytics Fixes Plan (June 2026)

This plan details the root cause analysis and step-by-step resolution steps for the **5 active fatal issues** identified in the Firebase Crashlytics reports for the Android application (`windowschedule-21145`).

---

## Issue Summary (Last 90 Days)

| Priority | Issue / Stacktrace | Events | Users | Category / Status |
| :--- | :--- | :---: | :---: | :--- |
| **High** | `cafe.adriel.voyager.androidx.AndroidScreenLifecycleOwner.emitOnStopEvents` | 57 | 22 | Navigation Lifecycle / Regressed |
| **High** | `androidx.compose.runtime.saveable.SaveableStateHolderImpl` (`LoginScreen:transition` duplicate key) | 31 | 10 | Compose Saveable State / Active |
| **Medium** | `cafe.adriel.voyager.androidx.AndroidScreenLifecycleOwner$registerLifecycleListener$observer$1.onStart` | 6 | 4 | Navigation Lifecycle / Regressed |
| **Medium** | `SubscriptionRepositoryImpl.getSubscription` (Flow transparency violation) | 4 | 2 | Coroutine Flow / Active |
| **Low** | `com.google.android.gms.auth.api.signin.internal.SignInHubActivity.onCreate` | 2 | 2 | Play Services Auth / Active |

---

## 1. Voyager Lifecycle Crashes (Issues 1 & 3)

### Stacktraces
- `java.lang.IllegalStateException - State is 'DESTROYED' and cannot be moved to STARTED in component cafe.adriel.voyager.androidx.AndroidScreenLifecycleOwner`
- `java.lang.IllegalStateException - State is 'DESTROYED' and cannot be moved to STARTED in component cafe.adriel.voyager.androidx.AndroidScreenLifecycleOwner$registerLifecycleListener$observer$1.onStart`

### Root Cause
These two issues stem from the same root bug in the **Voyager Navigation Library** (versions < `1.1.0-beta02`). 
Voyager's `AndroidScreenLifecycleOwner` tracks Compose screen lifecycles. When a screen is quickly popped or disposed, Android's framework lifecycle events (like `onStart` or `onStop`) can trigger out of order. The library attempts to transition the screen's lifecycle state to `STARTED` or calls `emitOnStopEvents` after the lifecycle has already reached `DESTROYED`. Jetpack Lifecycle constraints forbid moving a destroyed lifecycle back to an active state, resulting in a fatal `IllegalStateException`.

### Proposed Fix
#### Option A: Update Voyager library (Recommended)
Update Voyager to `1.1.0-beta02` (or the latest stable release) in your `gradle/libs.versions.toml` or `build.gradle` file:
```toml
# libs.versions.toml
voyager = "1.1.0-beta02" # or latest stable
```
This version includes a fix that checks if the screen lifecycle is already destroyed before executing state transitions.

#### Option B: Define a Custom Safe Lifecycle Owner (Workaround)
If upgrading the library is blocked, create a custom wrapper class to intercept and ignore lifecycle state changes when the state is `DESTROYED`:
```kotlin
// Create a wrapper that checks state before passing to lifecycleRegistry
class SafeAndroidScreenLifecycleOwner(
    private val delegate: AndroidScreenLifecycleOwner
) : ScreenLifecycleOwner by delegate {
    override val lifecycle: Lifecycle
        get() = if (delegate.lifecycle.currentState == Lifecycle.State.DESTROYED) {
            // Return a stub destroyed lifecycle
            SimpleDestroyedLifecycle()
        } else {
            delegate.lifecycle
        }
}
```

---

## 2. Compose SaveableStateHolder Key Reuse Crash (Issue 2)

### Stacktrace
- `java.lang.IllegalArgumentException - Key com.markduenas.windowschedule.presentation.screens.auth.LoginScreen:transition was used multiple times` in `androidx.compose.runtime.saveable.SaveableStateHolderImpl.SaveableStateProvider`

### Root Cause
This occurs when Jetpack Compose's state-saving mechanisms (`SaveableStateHolder`) attempt to register the key `com.markduenas.windowschedule.presentation.screens.auth.LoginScreen:transition` when it is already in use. 
This typically occurs under two conditions:
1. The user taps a transition trigger (like a login submit or back button) multiple times in rapid succession, initiating two overlapping transition animations of the same `LoginScreen`.
2. A transition/navigation block wrapper (e.g., Voyager's `SlideTransition` or `ScaleTransition`) caches the transition state using the screen key, and due to rapid navigation changes, the previous transition container is not fully disposed before the new one is created.

### Proposed Fix
1. **Debounce Navigation Actions:** Implement a click-debouncing mechanism (such as helper extension function `onClick` or a Compose state flag `isNavigating`) to prevent double-tapping buttons that trigger transition navigations:
   ```kotlin
   // Debounce Helper
   fun Modifier.clickableDebounced(
       enabled: Boolean = true,
       delayMillis: Long = 500L,
       onClick: () -> Unit
   ): Modifier = composed {
       var lastClickTime by remember { mutableStateOf(0L) }
       clickable(enabled = enabled) {
           val currentTime = System.currentTimeMillis()
           if (currentTime - lastClickTime > delayMillis) {
               lastClickTime = currentTime
               onClick()
           }
       }
   }
   ```
2. **Utilize Unique Navigation Keys:** If utilizing screen transitions, ensure the transition container is assigned a unique key (e.g., combining the screen key with a unique transition identifier or transaction ID) to prevent collisions in the `SaveableStateHolder`.

---

## 3. Flow Exception Transparency Violation (Issue 4)

### Stacktrace
- `java.lang.IllegalStateException - Flow exception transparency is violated: Previous 'emit' call has thrown exception kotlinx.coroutines.flow.internal.AbortFlowException: Flow was aborted, no more elements needed, but then emission attempt of value 'null' has been detected. Emission` in `SubscriptionRepositoryImpl.getSubscription`

### Root Cause
In Kotlin Coroutines, flows require exception transparency: a flow builder should not catch exceptions thrown by downstream collectors (like `AbortFlowException` which is thrown internally by downstream operators like `first()` or `take(1)`) and then attempt to emit subsequent values.
In `SubscriptionRepositoryImpl`, the flow is likely structured with an inline `try-catch` block:
```kotlin
// INCORRECT (Violates Transparency)
fun getSubscription(): Flow<Subscription?> = flow {
    try {
        val sub = apiService.fetchSubscription()
        emit(sub)
    } catch (e: Exception) {
        emit(null) // Crash! If AbortFlowException is caught here and we call emit
    }
}
```
If the downstream collector cancels the flow (throwing `AbortFlowException`), the `catch (e: Exception)` block catches it and tries to emit `null`, violating flow transparency rules and throwing the fatal `IllegalStateException`.

### Proposed Fix
Refactor the repository flow to use the built-in `.catch` operator on the flow chain, which handles flow cancellation and exception transparency rules safely, or re-throw cancellation exceptions explicitly:

#### Refactoring with `.catch` (Recommended):
```kotlin
// CORRECT
fun getSubscription(): Flow<Subscription?> = flow {
    val sub = apiService.fetchSubscription()
    emit(sub)
}.catch { e ->
    emit(null) // Safe: catch operator handles transparency automatically
}
```

#### Refactoring with Explicit Re-throw:
If using a `try-catch` inside the builder is absolutely necessary, re-throw `CancellationException` and its subclasses:
```kotlin
// CORRECT (Manual bypass)
import kotlinx.coroutines.CancellationException

fun getSubscription(): Flow<Subscription?> = flow {
    try {
        val sub = apiService.fetchSubscription()
        emit(sub)
    } catch (e: Exception) {
        if (e is CancellationException) throw e // Pass cancellation to downstream
        emit(null)
    }
}
```

---

## 4. Google Play Services Sign-in Hub Crash (Issue 5)

### Stacktrace
- `java.lang.NullPointerException - Attempt to invoke virtual method 'java.lang.Class java.lang.Object.getClass()' on a null object reference` in `com.google.android.gms.auth.api.signin.internal.SignInHubActivity.onCreate`

### Root Cause
This is a known, internal library issue within Google Play Services Auth dependency `com.google.android.gms:play-services-auth:20.7.0` (and similar `20.x.x` releases). 
When `SignInHubActivity` is launched with an empty intent (due to process recreation, system memory clearance, or a misconfigured sign-in intent builder), it attempts to extract credentials from a null extra object, leading to a fatal `NullPointerException`.

### Proposed Fix
Update the `play-services-auth` dependency in your Android app's dependency configuration (`gradle/libs.versions.toml` or `app/build.gradle`) to version `21.0.0` or higher, which includes internal checks resolving this recreation crash:
```toml
# libs.versions.toml
playServicesAuth = "21.1.0" # Update to latest stable
```

---

## Verification & Deployment Strategy
1. **Apply Dependency Updates:** Apply Voyager (`1.1.0-beta02`+) and Play Services Auth (`21.1.0`+) updates first.
2. **Refactor Repositories:** Scan repository files for internal flow `try-catch` blocks and replace them with standard `.catch {}` flow operators.
3. **Internal Testing:**
   - Execute regression tests on navigation (rapid back-and-forth navigations to verify Voyager and Compose transitions).
   - Simulate cancellation/disposal on Flows in unit tests to verify transparency.
4. **Gradual Staged Rollout:** Deploy the fixes in a staged rollout (e.g., 1% -> 5% -> 20% -> 100%) on Google Play to monitor crash rates on the Firebase Crashlytics dashboard.

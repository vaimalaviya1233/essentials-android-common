# Native Progressive Blur (Compose 1.13+)

Starting in **Compose 1.13.0-alpha03** (`androidx.compose.ui:ui:1.13.0-alpha03`), Jetpack Compose introduced the native `Modifier.blur { ... }` builder lambda API to support **progressive blurs** (e.g. gradient-based transitions, masks, and custom shader-driven blur falloffs) directly without needing manual custom AGSL jitter-sampling shaders or boilerplate `RenderEffect` management.

---

## 1. Overview & Comparison

| Feature | Legacy AGSL Custom Shader | Native `Modifier.blur { ... }` (1.13+) |
| :--- | :--- | :--- |
| **Compose Version** | Any (via Android 13+ `RuntimeShader`) | **1.13.0-alpha03+** |
| **API Surface** | Custom `RuntimeShader` + Jitter loop | Standard Compose UI Modifier |
| **Configuration** | Manual float uniforms & matrix calculations | Clean lambda DSL with `Brush`, gradients & falloffs |
| **Platform Compatibility** | Android 13+ (API 33+) | Android 12+ (API 31+ for RenderEffect blur) |
| **Performance** | Good (GPU fragment shader) | Native Skia / RenderNode pipeline optimization |

---

## 2. API Signature & Usage

The new progressive blur overload provides a builder scope (`BlurScope`) to configure spatial blur parameters using `BlurRadiusSpec`:

```kotlin
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.blur
import androidx.compose.ui.graphics.blur.BlurRadiusSpec
import androidx.compose.ui.graphics.blur.BlurStop
import androidx.compose.ui.unit.dp

// Example 1: Directional gradient-driven progressive blur
Modifier.blur {
    // Gradient between two radii
    radius = BlurRadiusSpec.verticalGradient(
        startRadius = 24.dp,
        endRadius = 0.dp
    )
}

// Example 2: Multi-stop gradient blur with custom falloffs
Modifier.blur {
    radius = BlurRadiusSpec.verticalGradient(
        listOf(
            BlurStop(fraction = 0.0f, radius = 24.dp),
            BlurStop(fraction = 0.4f, radius = 8.dp),
            BlurStop(fraction = 1.0f, radius = 0.dp),
        )
    )
}
```

### Applying Progressive Blur to System Bar Headers

A common design pattern in modern Material 3 Expressive apps is applying a progressive blur behind top app bars, status bars, or bottom navigation surfaces:

```kotlin
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.statusBars
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.blur
import androidx.compose.ui.graphics.blur.BlurRadiusSpec
import androidx.compose.ui.graphics.blur.BlurStop
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

@Composable
fun ProgressiveBlurHeader(
    modifier: Modifier = Modifier,
    blurRadius: Dp = 20.dp
) {
    val density = LocalDensity.current
    val statusBarHeight = with(density) { WindowInsets.statusBars.getTop(density).toDp() }
    val headerHeight = statusBarHeight + 64.dp

    Box(
        modifier = modifier
            .fillMaxWidth()
            .height(headerHeight)
            .blur {
                radius = BlurRadiusSpec.verticalGradient(
                    startRadius = blurRadius,
                    endRadius = 0.dp
                )
            }
    )
}
```

---

## 3. Directional Presets (Top / Bottom Scrims)

For ease of reuse across features, encapsulate common directional progressive blurs:

```kotlin
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.blur
import androidx.compose.ui.graphics.blur.BlurRadiusSpec
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

enum class ProgressiveBlurDirection {
    TOP,
    BOTTOM
}

fun Modifier.progressiveBlur(
    maxRadius: Dp = 24.dp,
    direction: ProgressiveBlurDirection = ProgressiveBlurDirection.TOP
): Modifier = this.blur {
    radius = when (direction) {
        ProgressiveBlurDirection.TOP -> BlurRadiusSpec.verticalGradient(
            startRadius = maxRadius,
            endRadius = 0.dp
        )
        ProgressiveBlurDirection.BOTTOM -> BlurRadiusSpec.verticalGradient(
            startRadius = 0.dp,
            endRadius = maxRadius
        )
    }
}
```

---

## 4. Device & Power-Saving Guards

As with any hardware blur effect, keep power-saving guardrails in place:

### Power Saving Mode
Disable blur during system battery saver to prevent unnecessary GPU overhead:

```kotlin
fun isPowerSaveMode(context: Context): Boolean {
    val powerManager = context.getSystemService(Context.POWER_SERVICE) as? PowerManager
    return powerManager?.isPowerSaveMode == true
}
```

---

## 5. Summary & Migration Checklist

- [ ] Update `androidx.compose.ui:ui` dependency to `1.13.0-alpha03` or newer.
- [ ] Replace custom AGSL shaders with the native `Modifier.blur { ... }` block where progressive falloffs or gradient masks are needed.
- [ ] Ensure `radius` is set to `0.dp` or modifier is conditionally bypassed when power-saving mode is enabled.
- [ ] Always check minimum SDK (`Build.VERSION.SDK_INT >= Build.VERSION_CODES.S`) for hardware blur support.

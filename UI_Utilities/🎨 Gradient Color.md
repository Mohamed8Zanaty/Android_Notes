
# Gradient Colors in Jetpack Compose

Jetpack Compose provides a flexible `Brush` API for creating various gradient effects. This allows for direct Kotlin implementation, making styling dynamic and readable.

## Standard Gradients

Here are common types of gradients you can use:

### Vertical Gradient

Transitions colors from top to bottom. Useful for screen or card backgrounds.

```kotlin
Brush.verticalGradient(
    colors = listOf(Color(0xFF0F1417), Color(0xFF242A32)),
    startY = 0f,
    endY = Float.POSITIVE_INFINITY
)
```

### Horizontal Gradient

Transitions colors from left to right.

```kotlin
Brush.horizontalGradient(
    colors = listOf(Color(0xFF0F1417), Color(0xFF242A32)),
    startX = 0f,
    endX = Float.POSITIVE_INFINITY
)
```

### Linear Gradient (Custom Direction)

Allows for gradients along any specified angle or direction.

```kotlin
Brush.linearGradient(
    colors = listOf(Color(0xFF0F1417), Color(0xFF242A32)),
    start = Offset(0f, 0f), // Top-left
    end = Offset(Float.POSITIVE_INFINITY, Float.POSITIVE_INFINITY) // Bottom-right
)
```

### Radial Gradient

Creates a circular blend of colors radiating outwards from a central point.

```kotlin
Brush.radialGradient(
    colors = listOf(Color(0xFF0F1417), Color(0xFF242A32)),
    center = Offset.Unspecified, // Center of the composable
    radius = Float.POSITIVE_INFINITY
)
```

### Sweep Gradient (Angular)

Sweeps colors around a central point, often used for circular indicators.

```kotlin
Brush.sweepGradient(
    colors = listOf(Color(0xFF0F1417), Color(0xFF242A32)),
    center = Offset.Unspecified // Center of the composable
)
```

## Advanced Mesh Gradients

For more sophisticated, fluid, and animated gradient effects, consider **mesh gradients**. These are defined by a grid of colored points (vertices) with colors smoothly interpolated between them. Libraries like `ComposeMeshGradient` leverage OpenGL ES for high-performance rendering of such effects, enabling dynamic and interactive backgrounds (e.g., Spotify-style lava lamps).

While `Brush` API gradients are excellent for most standard use cases, mesh gradients are suitable when you need intricate, animated, or highly dynamic visual experiences that go beyond simple linear or radial transitions.

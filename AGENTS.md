# AGENTS.md

Example Gradle project for **Jaylib** (`uk.co.electronstudio.jaylib:jaylib:6.0.+`), the JavaCPP/Bytedeco-based Java binding for raylib. Each example is a standalone translated raylib C demo. Built with Gradle 9.8.2 wrapper on OpenJDK 27 (9.8 is the first Gradle line supporting Java 27; older wrappers fail with "Unsupported class file major version 71" on this JDK). Uberjars via `com.gradleup.shadow` 9.6.1. No test sources yet.

To upgrade Gradle later: `./gradlew wrapper --gradle-version=X.Y.Z`.

## Commands

```bash
./gradlew run                                # runs Main.java
./gradlew run -Pmain=examples.HeightMap      # runs a specific example class
./gradlew run -Pmain=examples.CubicMap
./gradlew build                              # compile (JUnit is wired up but no tests exist)
./gradlew distZip                            # distributable zip; resources/ bundled, APP_HOME env var points to install dir
./gradlew shadowJar                          # uberjar
```

Graphical examples open an OS window and block until closed — don't try to run them headless or as an automated step; instead verify with compilation (`./gradlew compileJava` / `build`).

## Layout

- `src/main/java/Main.java` — default entry point (`application.mainClass` falls back to `'Main'`).
- `src/main/java/examples/` — self-contained examples, each with its own `main()`; run via `-Pmain=examples.ClassName`. They are templates, not library code — delete or edit freely.
- `src/main/java/examples/RLights/Light.java` — shader-light helper used by `ShadersBasicLighting`.
- `src/main/java/examples/shapes.java` — deliberate lowercase filename; leave as-is.
- `resources/` — PNG textures + GLSL shaders (`lighting.vs`/`.fs`), loaded at runtime with relative paths like `"resources/heightmap.png"`. Relative paths assume CWD is the project root (or the `APP_HOME` dir inside a distZip install).
- `gradle/` — Gradle wrapper. Don't modify wrapper files unless asked.

## Jaylib API style

- Static-import the raylib API and colors (matching the C API):
  - `import static com.raylib.Raylib.*;` — `InitWindow`, `BeginDrawing`, `DrawText`, constants like `CAMERA_PERSPECTIVE`
  - `import static com.raylib.Colors.*;` — `RAYWHITE`, `VIOLET`, `RED`, …
  - `com.raylib.Helpers` — convenience constructors like `newVector2(...)`, `newRectangle(...)`
- Structs are heap objects with fluent setters returning `this`:
  `new Vector3().x(18).y(16).z(18)`, `new Camera3D()._position(...).target(...).up(...).fovy(45)`
- Watch for raylib names that clash with Java: field setters keep a `_` prefix, e.g. `camera._position(...)`; "position" is overloaded (camera position vs material-map slot, e.g. `model.materials().maps().position(0).texture(t)`).
- Function-style names are preserved from C, even when they'd be unusual in Java (e.g. `Vector3Zero()`).
- Pointer types come from `org.bytedeco.javacpp` (`FloatPointer`, `IntPointer`): needed for `SetShaderValue(...)` uniforms, `GetShaderLocation`, and raw audio in `AudioRawStream`. Struct/array/enum return values are Java objects, not pointers.
- Shader locations pattern: `shader.locs().put(SHADER_LOC_VECTOR_VIEW, GetShaderLocation(shader, "viewPos"))`.

## Common game-loop skeleton

```java
import static com.raylib.Colors.RAYWHITE;
import static com.raylib.Colors.VIOLET;
import static com.raylib.Raylib.*;

InitWindow(800, 450, "Demo");
SetTargetFPS(60);
Camera3D camera = new Camera3D()._position(new Vector3().x(18).y(16).z(18))
        .target(new Vector3())
        .up(new Vector3().x(0).y(1).z(0))
        .fovy(45).projection(CAMERA_PERSPECTIVE);
while (!WindowShouldClose()) {
    UpdateCamera(camera, CAMERA_ORBITAL);
    BeginDrawing();
    ClearBackground(RAYWHITE);
    BeginMode3D(camera);   // 3D only; pair each Begin mode with End mode
    DrawGrid(20, 1.0f);
    EndMode3D();
    DrawText("Hello world", 190, 200, 20, VIOLET);
    DrawFPS(20, 20);
    EndDrawing();
}
CloseWindow();
```

## Conventions & notable details

- macOS `-XstartOnFirstThread` is injected in `build.gradle` (`applicationDefaultJvmArgs`, gated on a portable `System.getProperty('os.name')` check) — required by GLFW; don't remove.
- `application.mainClass = providers.gradleProperty('main').getOrElse('Main')` — running any new class with a `main()` requires no build change, just `-Pmain=fully.qualified.Name`. Don't revert to the deprecated `project.properties` map (removed in Gradle 10), and avoid internal Gradle APIs (e.g. `DefaultNativePlatform`) in build scripts.
- jaylib dependency is a dynamic version (`6.0.+`); pin an exact version if builds need to be reproducible.
- The commented-out staging repository in `build.gradle` is for pre-release jaylib testing; leave commented unless asked.
- New examples should follow the raylib C originals closely (globals become locals; C arrays of floats become `FloatPointer` — see `ShadersBasicLighting` and `RLights/Light.java`).
- `build.gradle` `distributions` block copies `resources/` into the zip (into the app home), which complements the relative-path loads.
- Adding tests requires creating `src/test/java/` (JUnit 5.7.0 deps are already declared); note unit-testing graphical raylib code in CI is impractical.

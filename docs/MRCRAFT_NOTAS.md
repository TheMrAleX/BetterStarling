# MrCraft - Notas de reconocimiento (Fase 1)

## a) Cómo se compila y firma el APK

**Script principal:** `packaging/android/build_apk.py` (Python 3)

**Flujo de compilación:**
1. `cargo ndk` compila `horizon-android` (cdylib) para el ABI objetivo (Rust profile `quick` por defecto, `fast` para releases)
2. `clang` del NDK compila los launchers Java nativos (`java_launcher.c` → `libstarling_java.so`, `libstarling_javac.so`) y copia `libc++_shared.so`
3. `tools/build_android_exporters.py` genera el `prepare-kit.zip` (exporters Java compilados en el host)
4. `aapt2` compila recursos (manifest, iconos, temas, colores) y genera `R.java`
5. `javac --release 8` + `d8` compila el código Java (`LauncherActivity`, `StarlingActivity`, `GameActivity`, `DiagnosticsActivity`, `WorkService`, providers, etc.) a `classes.dex`
6. `aapt2 link` genera `base.apk` con manifest, recursos y assets
7. Se añaden `classes.dex` y las bibliotecas nativas (`lib/<abi>/*.so`) al APK
8. `zipalign -P 16 -f 4` alinea el APK
9. `apksigner sign` firma con keystore (release: keystore de `HORIZON_ANDROID_KEYSTORE`; debug: keystore local `target/android/debug.keystore`, alias `horizon-debug`, password `android`)
10. `apksigner verify --print-certs` verifica la firma

**ABIs soportados:** `arm64-v8a` (default), `armeabi-v7a`, `x86`, `x86_64` (definidos en `TARGETS` en build_apk.py línea 55-60)

**Toolchain del workflow existente (`.github/workflows/android.yml`):**
- JDK: Temurin 25
- Android SDK: build-tools 35.0.0, platform android-36, NDK 28.2.13676358
- Rust: stable + targets para cada ABI, cargo-ndk 3.5.4
- Python: 3.12
- Perfil Cargo: `quick` (dev/PR), `fast` (release tag)
- Timeout: 150 min por ABI

**Firma:** Release usa secrets `STARLING_KEYSTORE_BASE64`, `STARLING_KEY_ALIAS`, `STARLING_STORE_PASSWORD`, `STARLING_KEY_PASSWORD` en environment `release`. Debug/Preview usa `STARLING_DEBUG_KEYSTORE_BASE64` o genera uno temporal.

---

## b) Activities/Services/Dialogs/Views del lado Android

| Componente | Archivo | Descripción | Proceso |
|---|---|---|---|
| **LauncherActivity** | `packaging/android/java/.../LauncherActivity.java` | Recibe archivos compartidos/abiertos (`.starling`, `.hmod`, `.hnmod`, `.zip`, packs, worlds), los copia al `import-inbox` del juego; maneja reinicios del juego (`AFTER` pid). **NO es LAUNCHER hoy** (ver abajo). | `:launcher` |
| **StarlingActivity** | `packaging/android/java/.../StarlingActivity.java` | `NativeActivity` extendida: storage folder, system bars, IME height, clipboard, file picker, browser, keystore, pointer capture, controller vibration, gyroscope, native mod permissions, build server dialog, restart, update install. **Es la Activity LAUNCHER real** (via activity-alias en manifest). | main |
| **GameActivity** | `packaging/android/java/.../GameActivity.java` | `extends StarlingActivity` — entry point del juego (manifest: `android.app.lib_name=horizon_android`, `android.app.func_name=starling_on_create`). | main |
| **DiagnosticsActivity** | `packaging/android/java/.../DiagnosticsActivity.java` | Ventana de sistema (sin cargar Rust/GPU): ver log, exportar ZIP diagnóstico, compartir, reintentar inicio. | `:diagnostics` |
| **WorkService** | `packaging/android/java/.../WorkService.java` | Foreground service (dataSync) para mantener proceso vivo durante sign-in Microsoft y descargas. Notificación "Starling at work". | main |
| **GameFolderProvider** | `packaging/android/java/.../GameFolderProvider.java` | DocumentsProvider para carpeta del juego en Files app (shared storage + app files). | `:documents` |
| **UpdateProvider** | `packaging/android/java/.../UpdateProvider.java` | FileProvider para APK descargado (installUpdate). | main |
| **HorizonImport** | `packaging/android/java/.../HorizonImport.java` | Importa worlds/packs de Horizon Android (dev.horizon.android) vía folder picker. | main |
| **TextMirror** | `packaging/android/java/.../TextMirror.java` | Editor de texto para IME (teclado en pantalla), sincroniza con `TextMirror` del juego. | main |
| **NativeMods** | `packaging/android/java/.../NativeMods.java` | Gestión de builds con native mods (stamp, chosen, permissions, build server). | main |
| **StartupRecovery** | `packaging/android/java/.../StartupRecovery.java` | Detección de inicio fallido previo, ofrece DiagnosticsActivity. | main |

**Activity LAUNCHER actual:** En `AndroidManifest.xml` líneas 97-105, un `activity-alias` llamado `io.github.juanluci.starling.StarlingActivity` apunta a `LauncherActivity` y tiene el `<intent-filter>` MAIN/LAUNCHER. La `LauncherActivity` real (líneas 48-95) **NO** tiene MAIN/LAUNCHER; solo maneja VIEW/SEND de archivos.

**Orientación:** `GameActivity` / `StarlingActivity` usan `android:screenOrientation="sensorLandscape"` (manifest línea 110).

---

## c) Toolkit de UI

**Lado Android (Java):** Solo APIs del framework (`android.view`, `android.graphics`, `Canvas`, `ValueAnimator`, `AlertDialog`, `LinearLayout`, `TextView`, `Button`, `GradientDrawable`, `RippleDrawable`, `SensorManager`, `Vibrator`, etc.). **NO hay AndroidX, NO hay Jetpack Compose, NO hay Material Components.**

**Lado motor (Rust/wgpu):** La UI del juego (menús, HUD, **Control Hub**, editor de controles, preparador, cuentas, etc.) se dibuja 100% en Rust con `wgpu`/`horizon-client-render`. El "Control Hub" (overlay morado #8B7CF6) está en `crates/horizon-client/src/android_control_ui.rs` (línea ~2793 "Horizon Control Hub") y se renderiza por el motor, **no es nativo Android**.

---

## d) Inventario de pantallas nativas Android

| Pantalla | Archivo | Qué muestra | Botones/Ajustes/Callbacks | Dibujada por |
|---|---|---|---|---|
| **Launcher (recibe archivo)** | `LauncherActivity.java` | Diálogo "Handing the file to Starling..." o pantalla negra "Opening the world with its mods…" | `play()` → inicia `GameActivity`; `receive()` copia a import-inbox | Android (framework) |
| **Diagnostics** | `DiagnosticsActivity.java` | Intro, botones: "View latest log", "Export diagnostic ZIP", "Share latest ZIP", "Try starting the game" (recovery), "Close" | `report(export)`, `showLog()`, `share()`, `retry` (start GameActivity) | Android (framework) |
| **WorkService notification** | `WorkService.java` | Notificación foreground "Starling at work" + línea de estado | `hold(context, reason, line, on)` desde `StarlingActivity.keepWorking()` | Android (NotificationManager) |
| **File picker (import)** | `StarlingActivity.pickFile()` | System `ACTION_OPEN_DOCUMENT` | `onActivityResult(PICK_FILE)` → `LauncherActivity.receive()` | Android (system picker) |
| **Horizon import** | `StarlingActivity.importFromHorizonAndroid()` | System folder picker (`HorizonImport.picker()`) | `onActivityResult(PICK_HORIZON)` → `HorizonImport.start()` | Android (system picker) |
| **Build server dialog** | `StarlingActivity.showBuildServerDialog()` | AlertDialog con EditText (URL), warning rojo, botones: "Use this server", "Use Starling's", "Cancel" | `NativeMods.chooseBuildServer()` | Android (AlertDialog) |
| **Mod permissions dialog** | `StarlingActivity.askModPermissions()` | AlertDialog: "A native mod needs a permission", lista mods + permiso, botones "Continue" / "Not now" | `requestPermissions()`, `NativeMods.declinePermissions()` | Android (AlertDialog) |
| **File access settings** | `StarlingActivity.openFileAccessSettings()` | System `ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION` | — | Android (system settings) |
| **Browser (Microsoft sign-in)** | `StarlingActivity.openLink()` | System browser `ACTION_VIEW` https:// | — | Android (system browser) |
| **Update install** | `StarlingActivity.installUpdate()` | System package installer (`UpdateProvider` URI) | — | Android (system installer) |

**Pantallas del motor Rust (NO se tocan en esta fase):**
- Preparación inicial (descarga Java, prepara assets) — `crates/horizon-client/src/preparer.rs` + UI en `android_control_ui.rs`
- Cuentas (Microsoft, Ely.by, Offline) — `crates/horizon-client/src/title/account.rs`
- Control Hub (Recommended/Controls/Aiming/Buttons/Advanced) — `android_control_ui.rs` ~línea 2793
- Editor de controles (botones flotantes) — `android_control_ui.rs`
- Ajustes del juego (video, audio, controles, etc.) — `options_ui.rs`
- Hub principal del juego (título, mundos, multijugador) — `title.rs`, `game.rs`

---

## e) Referencias al package viejo y updater

**Package actual:** `io.github.juanluci.starling` (en todos lados)

**Dónde aparece:**
- `build_apk.py`: línea 37 `APPLICATION_ID`, línea 215, 217, 235, 255 (custom-package), 318, 322
- `AndroidManifest.xml`: `package="@APPLICATION_ID@"`, activity names, authorities, shortcuts
- Todos los `.java`: `package io.github.juanluci.starling;`
- JNI en `android_lib.rs`: `Java_io_github_juanluci_starling_StarlingActivity_*`, `Java_io_github_juanluci_starling_TextMirror_*`, `Java_io_github_juanluci_starling_DiagnosticsActivity_makeReport`
- `shortcuts.xml`: `android:targetPackage="@APPLICATION_ID@"`, `targetClass`
- `starling_releases.rs`: línea 12 `REPOSITORIES = ["starling-client/starling", "JuanLuci/starling"]` — **el updater consulta estos dos repos en GitHub Releases API** (`latest_url` y `native_builds_url`)
- `build-server.txt`: `https://191-234-193-205.sslip.io` (build server por defecto para native mods)

**Nuevo package objetivo:** `com.themralex.mrcraft` (exacto)

---

## f) Orientación de pantalla

`GameActivity` / `StarlingActivity` en manifest: `android:screenOrientation="sensorLandscape"` (línea 110). `LauncherActivity` fuerza landscape en `showRestarting()` (línea 104) pero por defecto hereda del sistema. `DiagnosticsActivity` no fija orientación (vertical/horizontal según dispositivo).

**Conclusión:** La app es **horizontal (landscape)** para el juego; el launcher y diagnostics pueden rotar.
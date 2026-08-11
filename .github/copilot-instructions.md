# Copilot Instructions for OCXViewer

## Project Overview

OCXViewer is a JavaFX rich-client desktop application for inspecting and validating OCX files (Open Class 3D Exchange — a shipbuilding XML format). It is a diagnostic/inspection tool for OCX authors and consumers, **not** a generic 3D viewer or editor.

Java 21+, Maven, JPMS module `ocxviewer` (see `src/main/java/module-info.java`).

## Build & Test

JavaFX SDK must be downloaded manually from https://openjfx.io/ and extracted to:
- `binaries/windows/javafx-sdk-21.0.7/lib` (Windows)
- `binaries/linux/javafx-sdk-21.0.7/lib` (Linux)

```
mvn clean install                 # full build (also runs tests)
mvn test                          # tests only
mvn test -Dtest=OCXParserTest     # single test class
mvn test -Dtest=OCXParserTest#testReferences   # single test method
```

Notes:
- The build **generates the entire OCX data model** (`org.ocx_schema.v310.*` and `oasis.unitsml.*` packages) from `src/main/resources/OCX_Schema.xsd` via the `hisrc-higherjaxb-maven-plugin` (JAXB/XJC with `-Xinheritance` and `-XpropertyListener`). These classes only exist under `target/` — never hand-edit or commit generated model classes. Binding customizations live in `src/main/resources/global.xjb`.
- Tests run with `useModulePath=false` (unnamed module) and a working directory of `${java.io.tmpdir}`, but test code references data files via relative paths like `data/testLog4j2.xml` — run tests through Maven from the repo root.
- Test data (`.3docx` files) lives under `data/`.

Run the app (Windows; use `:` and `/` on Linux):

```
java --enable-native-access=javafx.graphics ^
  --module-path binaries\windows\javafx-sdk-21.0.9\lib;target;target\lib ^
  --add-modules javafx.controls,javafx.graphics,javafx.fxml,ocxviewer ^
  de.cadoculus.ocxviewer.Main --log data\log4j2.xml
```

## Architecture

Three source trees under `src/main/java`:
- `de.cadoculus.ocxviewer.*` — the application itself
- `net.jgeom.*` and `javax.media.j3d.*` — vendored NURBS/geometry libraries (third-party code, avoid modifying)

Application packages:
- `io/` — OCX file reading. `OCXParser` is the entry point for parsing (exposes a JavaFX `progressProperty()`); `OCXIO` handles JAXB marshalling/unmarshalling and **enforces the OCX 3.1.0 binding** for all reads/writes. `OCXIOReferenceResolver` resolves cross-references (`refId`) in the model after unmarshalling.
- `views/` — one `*Page` class per OCX concept (PanelPage, PlatePage, StiffenerPage, …). All detail pages extend `AbstractDataViewPage` (a `BorderPane` implementing the `Page` interface with `beforeShow`/`afterShow`/`beforeHide`/`afterHide`/`beforeClose` lifecycle). Drill-down subpages extend `AbstractDataViewSubPage`. `PageTree` builds the left-hand navigation tree.
- `event/` — simple event bus (`DefaultEventBus`, copied from the AtlantaFX sample app). Components communicate via bus events, not direct references. Event types: `HotkeyEvent`, `OpenEvent` (new OCX file opened), `NavigationEvent` (from nav tree), `SelectionEvent` (from detail view), `WindowEvent`, `ThemeEvent`.
- `actions/` — all actions extend `AbstractAction`. `ActionDispatcher.handleHotkeyEvent(HotkeyEvent)` dispatches hotkeys; for an action to be hotkey-triggered it must declare a static field, e.g. `public static final KeyCodeCombination KEYS = new KeyCodeCombination(KeyCode.O, KeyCombination.CONTROL_DOWN);`
- `geom/` — geometry tessellation of OCX curves/surfaces/brackets for display (uses the vendored `net.jgeom` NURBS code and `vecmath`). This is the only package with meaningful unit-test coverage.
- `models/` — small UI-facing records/enums (e.g. `BreadcrumbRecord`, `UnitRecord`).
- `logging/` — Log4j2 with a custom `ListBoxAppender` that feeds the in-app log view. Initialize logging in tests with `LoggerHelper.initLogging(new File("data/testLog4j2.xml"))`.

UI stack: AtlantaFX themes, Ikonli Material Design 2 icons, main layout in `src/main/resources/de/cadoculus/ocxviewer/main-view.fxml` with `MainController`.

## Conventions

- When adding a new detail page: extend `AbstractDataViewPage`, register it in `PageTree`, and fire/handle selection via the event bus — follow an existing page (e.g. `PlatePage`) as template.
- JPMS: new packages must be exported/opened in `module-info.java` (JavaFX FXML and JAXB need `opens`/`exports ... to`).
- Apache-2.0 license header on every new source file (see any existing file).
- Version info (git commit, build date shown in the About dialog) comes from Maven filtering of `mvn.properties` and the `git-commit-id-plugin` — don't hardcode versions.
- Help text uses AtlantaFX BBCode (`views/help.bbc`).

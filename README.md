# EMF Parsley JUnit 4 Examples

A small, standalone Maven project demonstrating how to test **EMF models and SWT/JFace components using plain JUnit 4**, with help from the testing library distributed with [EMF Parsley](https://www.eclipse.org/emf-parsley/).

**This project does not use the EMF Parsley application framework.** It uses `org.eclipse.emf.parsley.junit4`, a reusable test-support artifact, together with ordinary EMF, SWT, and JFace APIs. In particular, none of these examples requires launching an Eclipse workbench, an OSGi runtime, or SWTBot.

The examples are intentionally small: each one demonstrates a concrete testing boundary and can serve as a starting point for tests in other SWT/JFace applications.

## Requirements

- **JDK 21 or newer**, with Java 21 used as the compilation target.
- **Maven 3.9+** (the build uses ordinary Maven and Surefire, not Tycho).
- A graphical desktop session for the SWT/JFace tests. The EMF-only tests do not need a display.
- Network access to Maven repositories on the first build, unless dependencies are already cached.

### Running from the command line

```sh
mvn test
```

Run one example at a time:

```sh
mvn -Dtest=TreeViewerTest test
mvn -Dtest=TableViewerTest test
mvn -Dtest=DynamicEmfModelTest test
mvn -Dtest=ShellBasedTest test
mvn -Dtest=ControlBasedTest test
```

**macOS:** SWT requires its UI thread to be the first thread. The existing `macos` Maven profile is activated automatically on macOS and supplies `-XstartOnFirstThread` to Surefire via `argLine`. This matters for the SWT/JFace tests; the EMF-only tests are not UI-thread dependent.

**Linux:** Run the SWT/JFace tests in a graphical session with a functioning `DISPLAY` (or an appropriately configured virtual display). Tests that create SWT widgets will not run in a truly headless environment.

**Eclipse IDE:** Import the project with **File > Import > Existing Maven Projects** and run individual test classes with **Run As > JUnit Test**. On macOS, the JUnit launch configuration may need `-XstartOnFirstThread` under **Arguments > VM arguments** (Maven's Surefire setting is not automatically applied to Eclipse's own JUnit launches).

### Example overview

| Test | What it demonstrates | Needs SWT/display? | Uses EMF Parsley testing support? |
| --- | --- | --- | --- |
| `TreeViewerTest` | `DisplayHelperTestRule`, JFace tree content and labels, SWT `TreeItem` assertions | Yes | Yes |
| `TableViewerTest` | A real `TableViewer` with two columns, content/label providers, rendered cell values, and selection | Yes | Yes |
| `DynamicEmfModelTest` | Dynamic Ecore model, `EObject` values, and EMF change notifications | No | No; plain JUnit is sufficient |
| `EmfTreeViewerTest` | Dynamic EMF containment model in a JFace tree; structural and label updates from EMF notifications | Yes | Yes |
| `UiThreadTest` | Cross-thread `Display.syncExec`, UI-thread identity, and event-loop pumping | Yes | Yes |
| `WidgetEventsTest` | SWT listener dispatch and queued `Display.asyncExec` without simulated native input | Yes | Yes |
| `EmfDataBindingTest` | Bidirectional binding of an EMF attribute to an SWT `Text` in an explicit `DisplayRealm` | Yes | Yes |
| `ShellBasedTest` | Inherited `Display`/`Shell` fixture, `syncExec`, `syncExecVoid`, and `execAndFlushPendingEvents` | Yes | Yes |

The EMF-only test intentionally **does not** use the support library: the simplest faithful test for model-only behavior needs neither SWT nor a dedicated UI test fixture.

## How the SWT tests work

`DisplayHelperTestRule` manages an SWT `Display` and `Shell` for each test. The examples create and inspect real SWT/JFace widgets through `displayHelper.getShell()` and make ordinary JUnit assertions about the resulting state.

SWT widget access must take place on the thread that owns the display. These starter UI tests create and inspect their controls directly on the JUnit test thread under the rule's normal execution conditions. `UiThreadTest` demonstrates cross-thread `Display.syncExec`: a worker requests widget access, while the JUnit/UI thread dispatches the callback. It must not block on `join()` before dispatching the event loop.

Alternatively, `ShellBasedTest` derives from `AbstractEmfParsleyShellBasedTest`, which already installs the same rule and exposes `getShell()`, `getDisplay()`, `syncExec`, `syncExecVoid`, and `execAndFlushPendingEvents`. Do not declare an additional `@Rule` field in subclasses: the inherited rule manages fixture cleanup. The `syncExec` family runs operations on the SWT display thread; `execAndFlushPendingEvents` runs its supplier in the **calling thread** and then drains pending display events, so it is not a replacement for UI-thread marshalling.

The rule also handles fixture cleanup. `WidgetEventsTest` deliberately calls `notifyListeners` to exercise a listener, not to assert that a real mouse click works. These tests do not simulate user gestures, open Eclipse views, or rely on workbench services; tests for those contracts may need a different execution environment.

`EmfTreeViewerTest` deliberately uses a tiny, self-contained `ITreeContentProvider` with an `EContentAdapter` to bridge EMF notifications to the viewer. **This is not an EMF.Edit `AdapterFactoryContentProvider` test**: a generated EMF.Edit item-provider setup can perform notification-to-viewer updates for an application, but that requires additional provider infrastructure. The test illustrates the contract without it.

`EmfDataBindingTest` uses the SWT display realm so that observable values and UI updates run on the correct thread. It also explicitly disposes its `DataBindingContext`, which owns the binding subscriptions.

## Project layout

```text
pom.xml
README.md
src/test/java/io/github/lorenzobettini/example/
  TreeViewerTest.java
  TableViewerTest.java
  DynamicEmfModelTest.java
  EmfTreeViewerTest.java
  UiThreadTest.java
  WidgetEventsTest.java
  EmfDataBindingTest.java
  ShellBasedTest.java
```

There is no production application in this project. Its artifact is intended as an executable example suite.

## Further example ideas

The collection now includes examples of both SWT-oriented abstract classes. Potential future additions, **not implemented here**, include generated EMF.Edit item-provider tests, a focused example of `AbstractEmfParsleyTest` dependency-injection utilities, and test helpers for more complex event sequences.

Keep new tests independent of EMF Parsley's application framework; adding a more realistic client application would be a separate exercise.

## Scope and reproducibility

This project demonstrates **technical consumption** of the EMF Parsley JUnit 4 support artifact from a separate, ordinary Maven project. It does **not** establish independent third-party adoption or prove that a particular layered testing strategy improves cost or maintainability across projects.

The source intentionally uses no EMF Parsley application classes. You can inspect the resolved dependencies with:

```sh
mvn dependency:tree
```

The dependency tree may contain EMF and Eclipse platform libraries; that does not imply that the project launches the Eclipse workbench or uses the EMF Parsley application framework.

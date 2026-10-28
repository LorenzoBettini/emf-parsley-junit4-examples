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
mvn -Dtest=EmfEditTreeViewerTest test
mvn -Dtest=ShellBasedTest test
mvn -Dtest=ControlBasedTest test
mvn -Dtest=ViewerAssertionsTest test
mvn -Dtest=DefaultRealmTest test
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
| `EmfEditTreeViewerTest` | Generated Ecore EMF.Edit item providers, JFace tree hierarchy, and model-notification-driven updates | Yes | Yes |
| `UiThreadTest` | Cross-thread `Display.syncExec`, UI-thread identity, and event-loop pumping | Yes | Yes |
| `WidgetEventsTest` | SWT listener dispatch and queued `Display.asyncExec` without simulated native input | Yes | Yes |
| `EmfDataBindingTest` | Bidirectional binding of an EMF attribute to an SWT `Text` in an explicit `DisplayRealm` | Yes | Yes |
| `ShellBasedTest` | Inherited `Display`/`Shell` fixture, `syncExec`, `syncExecVoid`, and `execAndFlushPendingEvents` | Yes | Yes |
| `ControlBasedTest` | Inherited assertions for checkbox/text/combo and control states; programmatic EMF-object selection; `syncExecInRealm` | Yes | Yes |
| `ViewerAssertionsTest` | Inherited `assertTable` and `assertAllLabels` for complete real SWT/JFace viewer contents | Yes | Yes |
| `DefaultRealmTest` | `TestDefaultRealm` and Core Data Binding value-change notifications without SWT | No | Yes |

The EMF-only test intentionally **does not** use the support library: the simplest faithful test for model-only behavior needs neither SWT nor a dedicated UI test fixture.

## How the SWT tests work

`DisplayHelperTestRule` manages an SWT `Display` and `Shell` for each test. The examples create and inspect real SWT/JFace widgets through `displayHelper.getShell()` and make ordinary JUnit assertions about the resulting state.

SWT widget access must take place on the thread that owns the display. These starter UI tests create and inspect their controls directly on the JUnit test thread under the rule's normal execution conditions. `UiThreadTest` demonstrates cross-thread `Display.syncExec`: a worker requests widget access, while the JUnit/UI thread dispatches the callback. It must not block on `join()` before dispatching the event loop.

Alternatively, `ShellBasedTest` derives from `AbstractEmfParsleyShellBasedTest`, which already installs the same rule and exposes `getShell()`, `getDisplay()`, `syncExec`, `syncExecVoid`, and `execAndFlushPendingEvents`. Do not declare an additional `@Rule` field in subclasses: the inherited rule manages fixture cleanup. The `syncExec` family runs operations on the SWT display thread; `execAndFlushPendingEvents` runs its supplier in the **calling thread** and then drains pending display events, so it is not a replacement for UI-thread marshalling.

The rule also handles fixture cleanup. `WidgetEventsTest` deliberately calls `notifyListeners` to exercise a listener, not to assert that a real mouse click works. These tests do not simulate user gestures, open Eclipse views, or rely on workbench services; tests for those contracts may need a different execution environment.

`EmfTreeViewerTest` uses a small `ITreeContentProvider` and an `EContentAdapter` to handle notifications from a dynamic EMF model. `EmfEditTreeViewerTest` complements it by using Ecore's generated EMF.Edit item providers for tree structure and labels, forwarding their change notifications to the JFace viewer. Both tests run with ordinary JUnit and SWT, without an Eclipse workbench.

`EmfDataBindingTest` uses the SWT display realm so that observable values and UI updates run on the correct thread. It also explicitly disposes its `DataBindingContext`, which owns the binding subscriptions.

`ControlBasedTest` derives from `AbstractEmfParsleyControlBasedTest`, inheriting the shell-based fixture and reusable assertions for checkbox, label, text, combo, styles, and enabled/editable states. It also demonstrates changing a text control with `modifyText` and selecting an EMF object through the inherited selection helper. Its `syncExecInRealm` example establishes the SWT `DisplayRealm` while constructing and reading a JFace observable. The observable is explicitly disposed when the test is done. These helpers are useful when testing custom forms or controls without an Eclipse workbench.

`ViewerAssertionsTest` extends `AbstractEmfParsleyShellBasedTest` to demonstrate comparing all rendered rows of a JFace table and the complete expanded hierarchy of a JFace tree. The tree assertion uses two spaces of indentation per level, matching the support library's default representation. Both assertions operate on real widgets, not on the input collections.

`DefaultRealmTest` shows how `TestDefaultRealm` permits Core Data Binding observables to be tested without SWT. It also restores the previous realm in a `finally` block, so the default realm does not leak into another test. Use a real `DisplayRealm` instead when an observable depends on SWT widgets.

## Project layout

```text
pom.xml
README.md
src/test/java/io/github/lorenzobettini/example/
  TreeViewerTest.java
  TableViewerTest.java
  DynamicEmfModelTest.java
  EmfTreeViewerTest.java
  EmfEditTreeViewerTest.java
  UiThreadTest.java
  WidgetEventsTest.java
  EmfDataBindingTest.java
  ShellBasedTest.java
  ControlBasedTest.java
  ViewerAssertionsTest.java
  DefaultRealmTest.java
```

There is no production application in this project. Its artifact is intended as an executable example suite.

## Scope and reproducibility

This project demonstrates the EMF Parsley JUnit 4 support artifact in a separate, ordinary Maven project.

The source intentionally uses no EMF Parsley application classes. You can inspect the resolved dependencies with:

```sh
mvn dependency:tree
```

The dependency tree may contain EMF and Eclipse platform libraries; that does not imply that the project launches the Eclipse workbench or uses the EMF Parsley application framework.

# EMF Parsley JUnit 4 Examples

A small, standalone Maven project demonstrating how to test **EMF models and SWT/JFace components using plain JUnit 4**, with help from the testing library distributed with [EMF Parsley](https://www.eclipse.org/emf-parsley/).

**This project does not use the EMF Parsley application framework.** It uses `org.eclipse.emf.parsley.junit4`, a reusable test-support artifact, together with ordinary EMF, SWT, and JFace APIs. In particular, none of these examples requires launching an Eclipse workbench, an OSGi runtime, or SWTBot.

The examples are intentionally small: each one demonstrates a concrete testing boundary and can serve as a starting point for tests in other SWT/JFace applications.

## Requirements

- **JDK 21 or newer**, with Java 21 used as the compilation target.
- **Maven 3.9+** (the build uses ordinary Maven and Surefire, not Tycho).
- A graphical desktop session for the SWT/JFace tests. The EMF-only tests do not need a display.
- Network access to Maven repositories on the first build, unless dependencies are already cached.

The Maven project uses JUnit **4.13.2**, JFace **3.40.0**, EMF Ecore **2.42.0**, and the EMF Parsley JUnit support artifact **1.18.0**. These are independently versioned dependencies; see `pom.xml` for their exact coordinates.

### Running from the command line

```sh
mvn test
```

Run one example at a time:

```sh
mvn -Dtest=MyTreeViewerTest test
mvn -Dtest=TableViewerTest test
mvn -Dtest=DynamicEmfModelTest test
```

**macOS:** SWT requires its UI thread to be the first thread. The existing `macos` Maven profile is activated automatically on macOS and supplies `-XstartOnFirstThread` to Surefire via `argLine`. This matters for the SWT/JFace tests; the EMF-only tests are not UI-thread dependent.

**Linux:** Run the SWT/JFace tests in a graphical session with a functioning `DISPLAY` (or an appropriately configured virtual display). Tests that create SWT widgets will not run in a truly headless environment.

**Eclipse IDE:** Import the project with **File > Import > Existing Maven Projects** and run individual test classes with **Run As > JUnit Test**. On macOS, the JUnit launch configuration may need `-XstartOnFirstThread` under **Arguments > VM arguments** (Maven's Surefire setting is not automatically applied to Eclipse's own JUnit launches).

### Example overview

| Test | What it demonstrates | Needs SWT/display? | Uses EMF Parsley testing support? |
| --- | --- | --- | --- |
| `MyTreeViewerTest` | `DisplayHelperTestRule`, JFace tree content and labels, SWT `TreeItem` assertions | Yes | Yes |
| `TableViewerTest` | A real `TableViewer` with two columns, content/label providers, rendered cell values, and selection | Yes | Yes |
| `DynamicEmfModelTest` | Dynamic Ecore model, `EObject` values, and EMF change notifications | No | No; plain JUnit is sufficient |

The EMF-only test intentionally **does not** use the support library: the simplest faithful test for model-only behavior needs neither SWT nor a dedicated UI test fixture.

## How the SWT tests work

`DisplayHelperTestRule` manages an SWT `Display` and `Shell` for each test. The examples create and inspect real SWT/JFace widgets through `displayHelper.getShell()` and make ordinary JUnit assertions about the resulting state.

SWT widget access must take place on the thread that owns the display. These starter UI tests create and inspect their controls directly on the JUnit test thread under the rule's normal execution conditions. A future example will demonstrate cross-thread `syncExec` explicitly; do not move widget assertions onto an arbitrary background thread.

The rule also handles fixture cleanup. The tests do not simulate user gestures, open Eclipse views, or rely on workbench services; tests for those contracts may need a different execution environment.

## Project layout

```text
pom.xml
README.md
src/test/java/io/github/lorenzobettini/example/
  MyTreeViewerTest.java
  TableViewerTest.java
  DynamicEmfModelTest.java
```

There is no production application in this project. Its artifact is intended as an executable example suite.

## Suggested next examples

The following examples are **planned, not yet implemented**:

1. **`EmfTreeViewerTest`:** use a small EMF containment model and an EMF.Edit content/label provider to display an EMF hierarchy; change the model and verify that the viewer refreshes through model notifications. This will need EMF.Edit dependencies and careful adapter disposal.
2. **`UiThreadTest`:** demonstrate a background thread requesting UI access through `Display.syncExec`, safely pumping the event loop on the display thread, with no sleeps or racy timeouts.
3. **`EmfDataBindingTest`:** bind an SWT `Text` to an EMF attribute using an appropriate default `Realm`; verify changes in both directions and clean up binding resources. This will require EMF Data Binding dependencies.
4. **`WidgetEventsTest`:** demonstrate deterministic dispatch and observation of a focused SWT event without a full workbench or SWTBot.

Examples will be added in small increments so that each can be compiled and tested on both Linux and macOS.

## Scope and reproducibility

This project demonstrates **technical consumption** of the EMF Parsley JUnit 4 support artifact from a separate, ordinary Maven project. It does **not** establish independent third-party adoption or prove that a particular layered testing strategy improves cost or maintainability across projects.

The source intentionally uses no EMF Parsley application classes. You can inspect the resolved dependencies with:

```sh
mvn dependency:tree
```

The dependency tree may contain EMF and Eclipse platform libraries; that does not imply that the project launches the Eclipse workbench or uses the EMF Parsley application framework.

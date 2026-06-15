# Developer Documentation — Blockly2J Spielfeld Exercise

## 1. Project Overview

This is a Blockly2J exercise scaffold for creating programming exercises based on Blockly. It is designed for educational contexts (e.g., university courses, bootcamps) where students:

1. Write a solution in Java (or use Blockly) in the **solution** folder.
2. Write tests in the **tests** folder.
3. A **template** project provides the Blockly editor interface that students interact with.

The **parent** project orchestrates everything: it compiles the template, solution, and tests, then runs validation.

---

## 2. Architecture

```
parent/                          ← Orchestrates build + test execution
├── template/                    ← Blockly editor (student-facing)
│   ├── src/Main.java           ← Entry point; loads Blockly workspace
│   ├── src/Main.json           ← Pre-loaded Blockly XML blocks
│   ├── b2j-metadata.json       ← Tracks Java modifications
│   └── blockly-config.json     ← Blockly editor configuration
├── solution/                    ← Reference solution (hidden from students)
│   ├── src/Main.java           ← Expected solution implementation
│   ├── src/Main.json           ← Expected Blockly XML output
│   ├── b2j-metadata.json       ← Tracks Java modifications
│   └── blockly-config.json     ← Blockly editor configuration
└── tests/                       ← Test suite
    ├── test/b2j/test/          ← JUnit test classes
    └── test/b2j/wrappers/      ← Test wrapper classes (one per test method)
```

### Folder Responsibilities

| Folder      | Purpose                                                                                                                                                    |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `parent/`   | Root Gradle project. Compiles all sub-projects and runs tests via custom tasks (`testSolutionLocal`, `testTemplateLocal`, `testSolution`, `testTemplate`). |
| `template/` | The Blockly editor project. Contains the Blockly configuration, the loaded workspace XML, and the entry point. Students work with this project.            |
| `solution/` | The reference solution. Contains the expected `Main.java` output and expected `Main.json` (Blockly XML). Used for comparison during testing.               |
| `tests/`    | Maven-based test suite. Contains `TestManager`, test cases, and wrapper classes. Tests verify that the solution/template produce correct output.           |

---

## 3. Configuration Files

### 3.1 `b2j-metadata.json`

Location: `template/b2j-metadata.json` and `solution/b2j-metadata.json`

```json
{
  "javaModified": []
}
```

**Purpose:** Tracks which Java source files have been modified by the user (from Blockly-generated code). The `javaModified` array should list filenames of Java files that were auto-generated or modified during Blockly editing.

**Fields:**

| Field          | Type       | Description                                                                                                                                       |
| -------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| `javaModified` | `string[]` | Array of Java filenames that have been modified/created by the Blockly-to-Java generation process. Empty `[]` means no custom Java modifications. |

**When to populate:** When students write custom Java code (not just Blockly blocks) that integrates with the generated code, add those filenames here.

---

### 3.2 `blockly-config.json`

Location: `template/blockly-config.json` and `solution/blockly-config.json`

```json
{
  "main": true,
  "version": 2,
  "categories": ["main"],
  "blocks": {
    "main": true
  }
}
```

**Purpose:** Configures the Blockly editor interface — which blocks are visible, how categories are structured, and what the exercise is about.

**Fields:**

| Field        | Type       | Description                                                                          |
| ------------ | ---------- | ------------------------------------------------------------------------------------ |
| `main`       | `boolean`  | Enables the default block category.                                                  |
| `version`    | `int`      | Configuration format version. Must match the current schema version (currently `2`). |
| `categories` | `string[]` | Array of category names to show (e.g., `["main"]`).                                  |
| `blocks`     | `object`   | Block-level toggles — keys are category names, values are booleans.                  |

**Usage Notes:**
- The `version` field determines the schema — the project currently uses v2.
- The `categories` and `blocks` fields control which Blockly categories and blocks are visible to students.

---

### 3.3 `Main.json`

Location: `template/src/Main.json` and `solution/src/Main.json`

**Purpose:** Contains the **Blockly workspace configuration as JSON** — the actual blocks and categories that are loaded when the editor opens.

- **`template/src/Main.json`**: The blocks shown to students by default (the "starting state").
- **`solution/src/Main.json`**: The expected blocks the student should produce (the "target state").

**Format:** This is a JSON file that specifies the Blockly categories and blocks configuration.

**When editing:** Add or modify categories and blocks in this file to set up the exercise's starting configuration. The solution's `Main.json` defines what students should achieve.

---

## 4. Source Files

### 4.1 `template/src/Main.java`

The entry point for the template project. Uses `ch.vorburger.blocksjava.swing.Main` to provide the Blockly editor with:
- The Blockly workspace loaded from `Main.json`
- A "Run" button that compiles and executes the generated Java code
- Output display in the terminal panel

Key features:
- Uses `Main` to create the Blockly editor frame
- Handles Blockly-to-Java generation via `BlocklyUtils`
- Supports incremental compilation (`JavacRunner`) and full recompilation (`CompileJava`)
- Runs the compiled code in a separate JVM (`RunJava`)

### 4.2 `solution/src/Main.java`

The entry point for the solution project. Same structure as the template but with the expected/complete `Main.json` workspace.

---

## 5. Build System

### 5.1 Gradle (Parent Project)

**Location:** `parent/`

**Root Project (`build.gradle`):**
- Uses a custom Gradle setup with the Blockly Gradle plugin (`ch.vorburger.blocksjava.gradle.plugin`) for building template and solution sub-projects.
- Uses a custom Java Exec task (`mvnInstallTests`) to invoke Maven for compiling tests before running them.
- Defines custom Gradle tasks for Docker-based and local test execution:
  - `testSolutionDocker` — Tests the solution in a Docker container.
  - `testTemplateDocker` — Tests the template in a Docker container.
  - `testSolutionLocal` — Runs solution tests locally (without Docker).
  - `testTemplateLocal` — Runs template tests locally (without Docker).

**Subprojects:**
- `:template` — Compiles the Blockly editor project
- `:solution` — Compiles the solution project
- `:tests` — Compiles the Maven-based test project (invoked via Gradle `Exec` task)

### 5.2 Maven (Tests)

**Location:** `tests/`

The tests use Maven (not Gradle) for the test module. The parent Gradle build invokes Maven to compile tests before running them.

**Test Structure:**
- `test/b2j/test/` — JUnit test classes (`TestManager.java`, `Tests.java`, etc.)
- `test/b2j/wrappers/` — One wrapper class per test method

---

## 6. Test Framework

### 6.1 Test Architecture

Tests are JUnit 4 tests using Hamcrest matchers. The test infrastructure supports two assertion modes:

| Mode            | Description                                                      |
| --------------- | ---------------------------------------------------------------- |
| `Output`        | Compares stdout output against expected strings.                 |
| `Source`        | Compares generated Java source code against expected strings.    |
| `BlocklyBlocks` | Compares generated Blockly block structure against expected XML. |

### 6.2 Creating a New Test

1. **Create a test class** in `tests/test/b2j/test/` (e.g., `AdditionTest.java`).
2. **Create a wrapper class** in `tests/test/b2j/wrappers/` (e.g., `AdditionTest.java` with the same name).
3. Register the test in `tests/test/b2j/test/TestSettings.java`:
   ```java
   .withTest(AdditionTest.class)
   ```

**Example Test Class:**
```java
public class AdditionTest extends Tests {
    @Test
    public void testAddition() {
        // Define expected output
        expectedOutput().withOutputLine("...");
        
        // Or compare Blockly blocks
        // expectedBlocklyBlocks().withBlocks(
        //     "<xml>...</xml>"
        // );
    }
}
```

**Example Wrapper Class:**
```java
public class AdditionTest {
    public void testAddition() {
        Main.main();  // Calls the student's solution
    }
}
```

### 6.3 Test Runner

Tests are executed via the Gradle task `testSolution` or `testTemplate`:
- **Docker mode** (default): Runs tests inside isolated Docker containers via `docker-compose.yml`.
- **Local mode**: Runs tests directly on the host machine.

---

## 7. Running the Project

### 7.1 Local Development

```bash
# From the parent directory

# Run the template (Blockly editor)
cd template && ./gradlew run

# Run the solution
cd solution && ./gradlew run

# Build all sub-projects
cd parent && ./gradlew build
```

### 7.2 Running Tests (Local)

```bash
cd parent

# Build tests via Maven
./gradlew mvnInstallTests

# Run solution tests locally
./gradlew testSolutionLocal

# Run template tests locally
./gradlew testTemplateLocal
```

### 7.3 Running Tests (Docker)

```bash
cd parent

# Fix Gradle permissions (if needed)
sudo chown -R 1000:1000 .gradle

# Run solution tests in Docker
./gradlew testSolution

# Run template tests in Docker
./gradlew testTemplate
```

### 7.4 Viewing Test Results

```bash
# Open HTML test report in browser
xdg-open build/reports/tests/testTemplate/index.html
xdg-open build/reports/tests/testSolution/index.html
xdg-open build/reports/tests/testTemplateLocal/index.html
xdg-open build/reports/tests/testSolutionLocal/index.html
```

### 7.5 Using the VS Code Tasks

The workspace defines the following VS Code tasks (run via `Ctrl+Shift+P` → `Tasks: Run Task`):

| Task                         | Description                      |
| ---------------------------- | -------------------------------- |
| `run template`               | Runs the Blockly editor          |
| `run solution`               | Runs the reference solution      |
| `mvn install tests`          | Compiles the test suite          |
| `test template local`        | Runs template tests locally      |
| `test solution local`        | Runs solution tests locally      |
| `test template docker`       | Runs template tests in Docker    |
| `test solution docker`       | Runs solution tests in Docker    |
| `open template test results` | Opens test report in browser     |
| `open solution test results` | Opens test report in browser     |
| `fix gradle permissions`     | Fixes Gradle directory ownership |

---

## 8. Docker Compose

**Location:** `parent/docker-compose.yml`

Defines two services:
- `artemis-template` — Runs the template project in an isolated container.
- `artemis-solution` — Runs the solution project in an isolated container.

**Configuration:**
```yaml
services:
  artemis-template:
    volumes:
      - ./gradle:/home/blockly/.gradle
      - ..:/home/blockly/workspace
    command: ["bash", "-c", "..."]
    network_mode: "host"
```

The containers share Gradle cache and have access to the workspace files.

---

## 9. Creating a New Exercise

### Step-by-Step

1. **Modify the template workspace** (`template/src/Main.json`):
   - Define the starting blocks students see.
   - Use the Blockly editor to design and export the workspace.

2. **Define the solution** (`solution/src/Main.json`):
   - Set the expected Blockly blocks students should produce.

3. **Write the solution code** (`solution/src/Main.java`):
   - Implement the expected program logic (called by wrappers).

4. **Configure the exercise** (`template/blockly-config.json`):
   - Set the `description` to describe the exercise.
   - Configure block categories if needed.

5. **Create tests**:
   - Add test class in `tests/test/b2j/test/`.
   - Add wrapper class in `tests/test/b2j/wrappers/`.
   - Register in `TestSettings.java`.

6. **Update `b2j-metadata.json`** (if needed):
   - List any modified Java files in `javaModified`.

7. **Run and validate**:
   ```bash
   cd parent
   ./gradlew testSolutionLocal   # Verify solution passes tests
   ./gradlew testTemplateLocal   # Verify template passes tests
   ```

---

## 10. Troubleshooting

### Gradle Permission Issues
```bash
# If you see permission errors with Gradle wrapper
sudo chown -R 1000:1000 .gradle
```

### Docker Not Available
If Docker is not installed or not running, use local test tasks instead:
```bash
./gradlew testSolutionLocal
./gradlew testTemplateLocal
```

### Tests Not Finding Test Classes
Ensure tests are compiled:
```bash
./gradlew mvnInstallTests
```

### Template/Solution JARs Missing
Rebuild the sub-projects:
```bash
cd template && ./gradlew build
cd solution && ./gradlew build
```

---

## 11. Key Files Summary

| File                                    | Purpose                                                           |
| --------------------------------------- | ----------------------------------------------------------------- |
| `parent/build.gradle`                   | Root Gradle build; defines Concourse test runner and sub-projects |
| `parent/settings.gradle`                | Defines sub-projects: template, solution, tests                   |
| `parent/docker-compose.yml`             | Docker services for isolated test execution                       |
| `template/build.gradle`                 | Template project build; applies Blockly plugin                    |
| `solution/build.gradle`                 | Solution project build; applies Blockly plugin                    |
| `template/src/Main.java`                | Template entry point (Blockly editor)                             |
| `solution/src/Main.java`                | Solution entry point                                              |
| `template/src/Main.json`                | Starting Blockly workspace XML                                    |
| `solution/src/Main.json`                | Expected Blockly workspace XML                                    |
| `template/b2j-metadata.json`            | Tracks modified Java files (template)                             |
| `solution/b2j-metadata.json`            | Tracks modified Java files (solution)                             |
| `template/blockly-config.json`          | Blockly editor configuration                                      |
| `tests/pom.xml`                         | Maven build for test module                                       |
| `tests/test/b2j/test/TestManager.java`  | Test manager (runs all registered tests)                          |
| `tests/test/b2j/test/TestSettings.java` | Registers which test classes to run                               |
| `tests/test/b2j/test/Tests.java`        | Base class for test definitions                                   |
| `tests/test/b2j/wrappers/`              | One wrapper class per test method                                 |

---

## 12. Plugin Information

### Blockly Gradle Plugin

Both `template/build.gradle` and `solution/build.gradle` apply:
```groovy
plugins {
    id 'ch.vorburger.blocksjava.gradle.plugin' version '0.5.5'
}
```

The plugin:
- Generates Java code from Blockly XML (`Main.json`)
- Provides the `run` task for launching the Blockly editor
- Handles Blockly-to-Java compilation and execution

### Concourse Gradle Plugin

The parent project uses the Concourse plugin for test execution:
```groovy
plugins {
    id 'ch.vorburger.gradle.concourse' version '3.4.4'
}
```

The plugin orchestrates test execution across sub-projects with Docker support.

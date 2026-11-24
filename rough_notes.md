If that's the exit code: it printed 1 for a failed run, so the backend can rely on it. Now the Phase 1 code. Four Java files and one settings change, all inside `backend`.

---

## Step 1: settings

Open `backend/src/main/resources/application.yaml` and replace it with:

```yaml
spring:
  application:
    name: ptaf-runner

server:
  address: 127.0.0.1
  port: 8000

ptaf:
  project-path: "C:/Users/HussainM/IdeaProjects/ARGO_Connects_PTAF"
  maven-cmd: "C:/Program Files/JetBrains/IntelliJ IDEA 2026.2.0.1/plugins/maven-plugin/lib/maven3/bin/mvn.cmd"
  runner-class: "com.ptaf.runners.Mohammed_Test_Runner"
  runs-dir: "C:/Users/HussainM/IdeaProjects/PTAF-Runner/runs"
```

These are all the paths in one place, so nothing is hardcoded in the Java code.

---

## Step 2: `RunResult.java`

Right-click `com.ptaf.backend`, then **New → Java Class**, name it `RunResult`, and replace its contents with:

```java
package com.ptaf.backend;

// What one test run returns
public record RunResult(
        String tag,
        int exitCode,
        boolean passed,
        long durationSeconds,
        String logFile) {
}
```

---

## Step 3: `TestExecutor.java`

Same way, name it `TestExecutor`:

```java
package com.ptaf.backend;

// Runs one tagged test. LocalExecutor runs
// it on this laptop; a pipeline version can
// be added later without changing the rest.
public interface TestExecutor {

    RunResult run(String tag) throws Exception;
}
```

---

## Step 4: `LocalExecutor.java`

Name it `LocalExecutor`:

```java
package com.ptaf.backend;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;

// Runs a test on this laptop: the same
// commands we checked by hand.
@Component
public class LocalExecutor implements TestExecutor {

    private final Path project;
    private final String mavenCmd;
    private final String runnerClass;
    private final Path runsDir;

    public LocalExecutor(
            @Value("${ptaf.project-path}") String project,
            @Value("${ptaf.maven-cmd}") String mavenCmd,
            @Value("${ptaf.runner-class}") String runnerClass,
            @Value("${ptaf.runs-dir}") String runsDir) {
        this.project = Paths.get(project);
        this.mavenCmd = mavenCmd;
        this.runnerClass = runnerClass;
        this.runsDir = Paths.get(runsDir);
    }

    // synchronized = only one run at a time
    @Override
    public synchronized RunResult run(String tag)
            throws Exception {

        long start = System.currentTimeMillis();

        // Each run gets its own folder
        String stamp = LocalDateTime.now().format(
                DateTimeFormatter.ofPattern(
                        "yyyy-MM-dd_HH-mm-ss"));
        Path runDir = runsDir.resolve(stamp);
        Files.createDirectories(runDir);
        Path log = runDir.resolve("console.log");

        // 1. Compile tests and list libraries
        int build = exec(log, List.of(
                mavenCmd, "-q",
                "test-compile",
                "dependency:build-classpath",
                "-Dmdep.outputFile=target/runner-classpath.txt",
                "-Dmdep.includeScope=test"));

        if (build != 0) {
            return result(tag, build, start, log);
        }

        // 2. Put the classpath in a file
        //    (too long for a command line)
        String libs = Files.readString(project
                .resolve("target/runner-classpath.txt"))
                .trim();
        String cp = "target/test-classes;"
                + "target/classes;" + libs;

        Path argFile = runDir.resolve("java-args.txt");
        Files.writeString(argFile,
                "-cp \"" + cp.replace('\\', '/') + "\"");

        // 3. Run the runner with the tag
        String java = Paths.get(
                System.getProperty("java.home"),
                "bin", "java").toString();

        int exit = exec(log, List.of(
                java,
                "-Dcucumber.filter.tags=" + tag,
                "@" + argFile,
                "org.junit.runner.JUnitCore",
                runnerClass));

        return result(tag, exit, start, log);
    }

    // Runs a command inside the test project
    // and appends its output to the log file
    private int exec(Path log, List<String> command)
            throws Exception {

        ProcessBuilder pb = new ProcessBuilder(command);
        pb.directory(project.toFile());
        pb.redirectErrorStream(true);
        pb.redirectOutput(ProcessBuilder.Redirect
                .appendTo(log.toFile()));
        return pb.start().waitFor();
    }

    private RunResult result(String tag, int exit,
                             long start, Path log) {
        long secs = (System.currentTimeMillis()
                - start) / 1000;
        return new RunResult(tag, exit, exit == 0,
                secs, log.toString());
    }
}
```

---

## Step 5: `RunController.java`

Name it `RunController`:

```java
package com.ptaf.backend;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.regex.Pattern;

// POST /api/runs?tag=@testing  runs one test
@RestController
@RequestMapping("/api/runs")
public class RunController {

    // Only a single tag like @RT_51-53.
    // Blocks anything else from reaching
    // the command line.
    private static final Pattern TAG =
            Pattern.compile("@[A-Za-z0-9_\\-]+");

    private final TestExecutor executor;

    public RunController(TestExecutor executor) {
        this.executor = executor;
    }

    @PostMapping
    public ResponseEntity<?> run(
            @RequestParam String tag) throws Exception {

        if (!TAG.matcher(tag).matches()) {
            return ResponseEntity.badRequest().body(
                    "Invalid tag. Use one tag like @RT_51-53");
        }
        return ResponseEntity.ok(executor.run(tag));
    }
}
```

---

## Step 6: keep run logs out of Git

The logs can contain customer data from Navigator and Connects, so they must never be committed. Right-click the **`PTAF-Runner`** root, then **New → File**, name it `.gitignore`, and add:

```
runs/
```

---

## Step 7: test it tonight

Navigator is down, so the test will fail. That's fine: it proves the plumbing works end to end.

1. Restart `BackendApplication` (stop it, then ▶ again).
2. In a PowerShell terminal, run:

```powershell
Invoke-RestMethod -Method Post "http://127.0.0.1:8000/api/runs?tag=@testing" -TimeoutSec 1800
```

3. It waits while the test runs, a few minutes. Expected result:

```
tag             : @testing
exitCode        : 1
passed          : False
durationSeconds : ...
logFile         : C:\...\PTAF-Runner\runs\2026-...\console.log
```

4. Open that `console.log`. It should contain the same output you saw in the terminal.

Tomorrow, with Navigator up, run the same command and you should get `passed : True`.

I couldn't compile this here, so send me a screenshot if anything shows red or the call returns an error.

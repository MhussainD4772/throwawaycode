# Copilot catalog setup

Here's the complete setup: three files, one run, then a check.

- **`.github/copilot-instructions.md`:** the rules. Copilot reads this automatically.
- **`CatalogGenerator.java`:** reads your real steps and locators from the code.
- **`docs/step-catalog.md` and `docs/locator-catalog.md`:** written by the generator. The step list is also copied into the instructions file, so Copilot always has it.

---

## Step 1: create the instructions file

1. Right-click the project root (`ARGO_Connects_PTAF`), then **New → Directory**, and name it `.github`.
2. Right-click `.github`, then **New → File**, and name it `copilot-instructions.md`.
3. Paste this and save:

```markdown
# ARGO_Connects_PTAF – Copilot instructions

## Project
- End-to-end UI automation for ARGO Connects
  and Navigator.
- Java, Cucumber (Gherkin), Playwright.
- Built on the FNB-ETAF framework jar
  (com.fnb_ptaf:fnb_ptaf, package com.ptaf).
- Feature files: src/test/resources/features
- Locators: YAML files with an "elements:" root
  (client_search_page.yml, navigator_page.yml,
  document_manger_page.yml, end_session_page.yml)
- Step definitions: com.ptaf.stepdefinitions
- Helpers: com.ptaf.utils

## How to answer
- Give the exact YAML line and the exact
  Gherkin step, ready to paste.
- Keep code lines short (I read on a small
  screen).
- Never put real customer data (SSNs, account
  numbers) in code, YAML or examples.

## Steps
- Use ONLY steps from the generated Step catalog
  at the end of this file, word for word.
  {arg} = page or locator name,
  {value} = quoted value, {n} = number.
- Never invent a step. If none fits, say so and
  write a new step definition in the same style.

## Locators
- All existing locators are listed in
  docs/locator-catalog.md. Before creating a new
  key, ask me to attach that file (or the page's
  YAML) and reuse an existing key if one fits.

## Locator format (YAML)
- One line per locator under the page key:
  key_name: "xpath_//..."
- Prefixes: xpath_, ID_, css_, LinkText_,
  text_, placeholder_. Prefix + underscore is
  required (xpath_ not xpath).
- Name keys after the field: *_flt for fields,
  *_btn for buttons, *_nav_flt for Navigator.

## Locator rules
- Anchor on stable things: IDs, label text,
  header text. Never on values that change
  (names, addresses, account numbers).
- Avoid page-wide indexes like [1] [2]; they
  shift when rows are added. Scope them inside
  a stable container instead.
- Label/value pairs in tables:
  //td[normalize-space()='Label:']
    /following-sibling::td[1]
- Field rows (e.g. Codes page):
  (//tr[.//*[normalize-space()='Field']])[last()]
    /td[normalize-space()][2]
- normalize-space does NOT remove &nbsp;. If a
  cell may be a spacer, target the whole row
  and use a contain check.
- Common XPath typos: parent::td needs two
  colons; brackets and quotes must balance.
- Test a locator in DevTools: click inside the
  Elements tree, press Ctrl+F, paste the XPath
  (without xpath_). Must show 1 of 1. The bottom
  "Search" tab does NOT support XPath. For
  elements in an iframe, right-click the element
  > Inspect first.

## Frames: which step wording to use
Connects:
- main frame: main application area
- second frame: frame inside main frame
- pop-up frame: pop-up dialogs
- warning-box frame: warning/confirm boxes
- header frame: page header
- DueDiligenceForm frame: due diligence form

Navigator:
- "on page navigator_page": top-level page
  (customer search box)
- "navigator frame": Main frame
  (Relationships tab etc.)
- "navigator bottom frame": Main > bottom
  frame (Customer Inquiry summary)
- "navigator page": the Name Inquiry POPUP
  window, opened by clicking clint_info_link.
  These steps wait for that popup; if it is not
  open they time out (waitForPopup error).

Rule: checks on a page must use the same frame
wording as the click that opened that page.

## contain vs has value
- contain: visible text (labels, cells).
  Case-sensitive, partial match.
- has value: inputs and dropdowns only. Reads
  the value attribute; for dropdowns that is
  the code (e.g. "US", not "United States").

## Navigator display formats
Navigator often shows data differently from
what was typed in Connects. Expected values
must match Navigator's format:
- Names: UPPER CASE (AUTO CUSTOMER)
- DOB on Customer Inquiry: Jan 01, 1998
- ID type: DRIVER'S LICENSE [A]
  -> expect DRIVER'S LICENSE
- Issued by: [OH] -> expect OH
- Phone: 773-302-6848 (with dashes)
- Email: as entered (case kept)
- Codes page dates: MM/dd/yyyy

## Reading failures
The report shows an "FNB-ETAF UI ACTION
FAILURE" block with Locator key, Frame path,
Match count and Cause message.
- Text mismatch: compare Expected vs Actual.
  Usually format or case (see formats above).
  Actual empty: locator hits an empty cell.
- Match count 0 / element null: wrong locator
  or wrong frame.
- waitForPopup / Timeout 30000ms on a
  "navigator page" step: the popup is not
  open; use navigator frame or navigator
  bottom frame wording.
- Step passes when it should fail: locator may
  be matching a header or template row.

## Test data
- No hardcoded dynamic data. Values that change
  per run (tax ID, address, account numbers,
  tokens) are generated or read from Excel.
- Static values go in the Examples table.
```

---

## Step 2: create the generator

1. Right-click `src/test/java/com/ptaf/utils`, then **New → Java Class**, and name it `CatalogGenerator`.
2. Replace everything in it with:

```java
package com.ptaf.utils;

import java.io.File;
import java.io.IOException;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDate;
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.TreeMap;
import java.util.TreeSet;
import java.util.jar.JarEntry;
import java.util.jar.JarFile;
import java.util.jar.Manifest;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import java.util.stream.Stream;

// Builds the Copilot catalogs from the code.
// Run with the green ▶ next to main whenever
// steps or locators change.
public class CatalogGenerator {

    private static final Path ROOT =
            Paths.get(System.getProperty("user.dir"));

    private static final Path STEP_FILE =
            ROOT.resolve("docs/step-catalog.md");

    private static final Path LOCATOR_FILE =
            ROOT.resolve("docs/locator-catalog.md");

    private static final Path INSTRUCTIONS =
            ROOT.resolve(
                ".github/copilot-instructions.md");

    private static final Path RESOURCES =
            ROOT.resolve("src/test/resources");

    private static final String START =
            "<!-- STEP-CATALOG:START -->";

    private static final String END =
            "<!-- STEP-CATALOG:END -->";

    private static final Set<String> STEP_TYPES =
            Set.of(
                "io.cucumber.java.en.Given",
                "io.cucumber.java.en.When",
                "io.cucumber.java.en.Then",
                "io.cucumber.java.en.And",
                "io.cucumber.java.en.But");

    public static void main(String[] args)
            throws Exception {

        Files.createDirectories(
                STEP_FILE.getParent());

        Map<String, Set<String>> steps = findSteps();
        String stepText = renderSteps(steps);

        Files.writeString(STEP_FILE,
                "# Step catalog\n\n" + header()
                + stepText,
                StandardCharsets.UTF_8);

        int locators = writeLocators();
        updateInstructions(stepText);

        int stepCount = steps.values().stream()
                .mapToInt(Set::size).sum();

        System.out.println("Steps: " + stepCount);
        System.out.println("Locators: " + locators);
        System.out.println("Wrote " + STEP_FILE);
        System.out.println("Wrote " + LOCATOR_FILE);
    }

    private static String header() {
        return "Generated " + LocalDate.now()
                + " by CatalogGenerator."
                + " Do not edit by hand.\n\n";
    }

    // ---------- Steps ----------

    // Finds every Cucumber step on the classpath
    // (your project and the framework jar)
    private static Map<String, Set<String>>
            findSteps() throws Exception {

        Map<String, Set<String>> result =
                new TreeMap<>();
        ClassLoader loader =
                CatalogGenerator.class
                        .getClassLoader();

        for (String name : findClassNames()) {
            try {
                Class<?> cls = Class.forName(
                        name, false, loader);

                for (Method m :
                        cls.getDeclaredMethods()) {
                    for (Annotation a :
                            m.getAnnotations()) {

                        String type = a.annotationType()
                                .getName();
                        if (!STEP_TYPES.contains(type)) {
                            continue;
                        }
                        Object value = a.annotationType()
                                .getMethod("value")
                                .invoke(a);

                        String group = cls.getName()
                                .replace("com.ptaf.", "");

                        result.computeIfAbsent(group,
                                k -> new TreeSet<>())
                              .add(readable(
                                String.valueOf(value)));
                    }
                }
            } catch (Throwable t) {
                // class can't load: skip it
            }
        }
        return result;
    }

    // Lists all com.ptaf classes in folders
    // and jars on the classpath
    private static List<String> findClassNames()
            throws Exception {

        List<String> names = new ArrayList<>();
        Deque<Path> todo = new ArrayDeque<>();
        Set<Path> seen = new HashSet<>();

        for (String e : System
                .getProperty("java.class.path")
                .split(File.pathSeparator)) {
            if (!e.isBlank()) {
                todo.add(Paths.get(e));
            }
        }

        while (!todo.isEmpty()) {
            Path p = todo.poll()
                    .toAbsolutePath().normalize();

            if (!seen.add(p) || !Files.exists(p)) {
                continue;
            }

            if (Files.isDirectory(p)) {
                try (Stream<Path> files =
                             Files.walk(p)) {
                    files.map(f -> p.relativize(f)
                                    .toString()
                                    .replace('\\', '/'))
                         .filter(CatalogGenerator
                                 ::isPtafClass)
                         .forEach(n -> names.add(
                                 toClassName(n)));
                }
            } else if (p.toString()
                    .endsWith(".jar")) {

                try (JarFile jar =
                             new JarFile(p.toFile())) {

                    jar.stream()
                       .map(JarEntry::getName)
                       .filter(CatalogGenerator
                               ::isPtafClass)
                       .forEach(n -> names.add(
                               toClassName(n)));

                    // IntelliJ may pack the
                    // classpath into a jar manifest
                    Manifest mf = jar.getManifest();
                    String extra = mf == null ? null
                            : mf.getMainAttributes()
                                .getValue("Class-Path");

                    if (extra != null) {
                        for (String u : extra.trim()
                                .split("\\s+")) {
                            try {
                                todo.add(Paths.get(
                                        new URI(u)));
                            } catch (Exception ignored) {
                            }
                        }
                    }
                }
            }
        }
        return names;
    }

    private static boolean isPtafClass(String n) {
        return n.startsWith("com/ptaf/")
                && n.endsWith(".class")
                && !n.contains("$");
    }

    private static String toClassName(String n) {
        return n.substring(0, n.length() - 6)
                .replace('/', '.');
    }

    // Turns the step regex into readable text
    private static String readable(String regex) {
        String s = regex;
        if (s.startsWith("^")) {
            s = s.substring(1);
        }
        if (s.endsWith("$")) {
            s = s.substring(0, s.length() - 1);
        }
        return s.replace("\\\"", "\"")
                .replace("\"(.*?)\"", "\"{value}\"")
                .replace("\"(.*)\"", "\"{value}\"")
                .replace("(.*?)", "{arg}")
                .replace("(.*)", "{arg}")
                .replace("(\\d+)", "{n}");
    }

    private static String renderSteps(
            Map<String, Set<String>> steps) {

        StringBuilder sb = new StringBuilder();
        sb.append("{arg} = page/locator/name, ")
          .append("{value} = quoted value, ")
          .append("{n} = number.\n");

        for (Map.Entry<String, Set<String>> e :
                steps.entrySet()) {
            sb.append("\n### ")
              .append(e.getKey()).append("\n");
            for (String s : e.getValue()) {
                sb.append("- ").append(s)
                  .append("\n");
            }
        }
        return sb.toString();
    }

    // ---------- Locators ----------

    // Reads every YAML file with an
    // "elements:" root under src/test/resources
    private static int writeLocators()
            throws IOException {

        Pattern line = Pattern.compile(
                "^(\\s*)([^\\s#][^:]*?):\\s*(.*?)\\s*$");

        StringBuilder sb = new StringBuilder(
                "# Locator catalog\n\n" + header());
        int count = 0;

        List<Path> files;
        try (Stream<Path> s =
                     Files.walk(RESOURCES)) {
            files = s.filter(f -> f.toString()
                            .matches(".*\\.ya?ml$"))
                     .sorted()
                     .toList();
        }

        for (Path f : files) {
            String text = new String(
                    Files.readAllBytes(f),
                    StandardCharsets.UTF_8)
                    .replace("\uFEFF", "");
            String[] lines = text.split("\\R");

            boolean isLocatorFile = false;
            for (String l : lines) {
                if (l.startsWith("elements:")) {
                    isLocatorFile = true;
                    break;
                }
            }
            if (!isLocatorFile) {
                continue;
            }

            sb.append("\n## ")
              .append(ROOT.relativize(f).toString()
                      .replace('\\', '/'))
              .append("\n");

            for (String l : lines) {
                if (l.trim().startsWith("#")) {
                    continue;
                }
                Matcher m = line.matcher(l);
                if (!m.matches()
                        || m.group(1).isEmpty()) {
                    continue;
                }
                String key = m.group(2).trim();
                String value = m.group(3);

                if (value.isEmpty()) {
                    sb.append("\n### ")
                      .append(key).append("\n");
                } else {
                    sb.append("- ").append(key)
                      .append(": ")
                      .append(unquote(value))
                      .append("\n");
                    count++;
                }
            }
        }

        Files.writeString(LOCATOR_FILE,
                sb.toString(),
                StandardCharsets.UTF_8);
        return count;
    }

    private static String unquote(String v) {
        if (v.length() >= 2
                && ((v.startsWith("\"")
                     && v.endsWith("\""))
                 || (v.startsWith("'")
                     && v.endsWith("'")))) {
            return v.substring(1, v.length() - 1);
        }
        return v;
    }

    // ---------- Instructions ----------

    // Puts the step list at the end of
    // copilot-instructions.md, replacing the
    // previous generated block
    private static void updateInstructions(
            String stepText) throws IOException {

        if (!Files.exists(INSTRUCTIONS)) {
            System.out.println("No "
                    + INSTRUCTIONS + ", skipped");
            return;
        }

        String content = Files.readString(
                INSTRUCTIONS,
                StandardCharsets.UTF_8);

        String block = START + "\n"
                + "## Step catalog (generated)\n"
                + "Use ONLY these steps, "
                + "word for word.\n"
                + stepText + END;

        int s = content.indexOf(START);
        int e = content.indexOf(END);

        if (s >= 0 && e > s) {
            content = content.substring(0, s)
                    + block
                    + content.substring(
                        e + END.length());
        } else {
            content = content.stripTrailing()
                    + "\n\n" + block + "\n";
        }

        Files.writeString(INSTRUCTIONS, content,
                StandardCharsets.UTF_8);
    }
}
```

It reads the steps from the compiled classes, not the source files. So commented-out steps are skipped automatically, and the framework jar's steps are included too.

---

## Step 3: run it

1. Click the green **▶** next to `main`, then **Run 'CatalogGenerator.main()'**.
2. The console should print something like:

```text
Steps: 450
Locators: 300
Wrote ...\docs\step-catalog.md
Wrote ...\docs\locator-catalog.md
```

3. Check:
   - a `docs` folder in the project root, with both files
   - the end of `.github/copilot-instructions.md` now has a **Step catalog (generated)** section

**If it says `Steps: 0`**, the run used a shortened classpath that the generator couldn't read. Open the run configuration, choose **Modify options → Shorten command line → @argfile (Java 9+)**, and run again.

---

## Step 4: check Copilot uses it

Open a **new** Copilot chat in Ask mode and ask:

1. *"Which step do I use to check text on the Customer Inquiry summary in Navigator?"* It should give the exact **navigator bottom frame** contain step, word for word from the catalog.
2. *"Which frame wording for a field on the Name Inquiry popup?"* It should say **navigator page**.

**If it answers generically**, it isn't reading the file. Look in **Settings → Tools → GitHub Copilot** for a custom instructions option and turn it on.

**If it gets the rules right but misses steps**, the file may be too long for it. In that case, delete the generated block (everything between the two `STEP-CATALOG` markers), and attach `docs/step-catalog.md` to the chat when you need steps.

---

## Step 5: use it day to day

- **Failures:** paste a screenshot of the FNB-ETAF failure block and ask what's wrong.
- **New locators:** send a DevTools screenshot and attach that page's YAML (type `#` in the chat to pick a file), so it reuses existing keys.
- **New steps:** after you add a step definition or locators, rerun `CatalogGenerator`.

---

## Step 6: commit and share

Commit these four files:

- `.github/copilot-instructions.md`
- `src/test/java/com/ptaf/utils/CatalogGenerator.java`
- `docs/step-catalog.md`
- `docs/locator-catalog.md`

Everyone on the project then gets the same Copilot setup. Whoever adds steps reruns the generator before committing.

I couldn't compile or run this here, so Step 3 is the real test. If anything shows red or the counts look wrong, send me a screenshot.

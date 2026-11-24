$mvn = "C:\Program Files\JetBrains\IntelliJ IDEA 2026.2.0.1\plugins\maven-plugin\lib\maven3\bin\mvn.cmd"


& $mvn -q test-compile dependency:build-classpath "-Dmdep.outputFile=target/runner-classpath.txt" "-Dmdep.includeScope=test"


$cp = "target\test-classes;target\classes;" + (Get-Content target\runner-classpath.txt -Raw).Trim()
java "-Dcucumber.filter.tags=@testing" -cp $cp org.junit.runner.JUnitCore com.ptaf.runners.Mohammed_Test_Runner


$LASTEXITCODE

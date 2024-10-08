Cucumber JUnit Platform Engine
==============================

* allows, about Cucumber scenarios -- via -- JUnit (5) Platform
  * discovering
  * selecting
  * executing  
* Maven
  * add | "pom.xml"

    ```xml
    <dependency>
       <groupId>io.cucumber</groupId>
       <artifactId>cucumber-junit-platform-engine</artifactId>
       <version>${cucumber.version}</version>
       <scope>test</scope>
    </dependency>
    ```

## Surefire and Gradle workarounds

* 👁️ discovery of NON-class based tests, NOT yet supported | Maven Surefire and Gradle 👁️
  * gradle
    * [gradle/#4773](https://github.com/gradle/gradle/issues/4773)
  * surefire
    * [SUREFIRE-1724](https://issues.apache.org/jira/browse/SUREFIRE-1724)
  * workarounds
    * [JUnit Platform Suite Engine](https://junit.org/junit5/docs/current/user-guide/#junit-platform-suite-engine)
    * [JUnit Platform Console Launcher](https://junit.org/junit5/docs/current/user-guide/#running-tests-console-launcher)
    * [Gradle Cucumber-Companion](https://github.com/gradle/cucumber-companion) -- plugins for -- Gradle and Maven
    * [Cucable](https://github.com/trivago/cucable-plugin) plugin -- for -- Maven

### Workaround -- via -- JUnit Platform Suite Engine

* JUnit Platform Suite Engine
  * allows
    * running Cucumber -- Check [Suites with different configurations](#suites-with-different-configurations) -- 
* set `cucumber.junit-platform.naming-strategy=long` as configuration
  parameter
  * -> improve the readability of the reports / feature name -- is part of the -- test name  
  * Reason: 🧠 Surefire and Gradle reports syntax `<Class Name> - <Method Name>`-> hard to read 🧠  

#### Maven

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <properties>
            <configurationParameters>
                cucumber.junit-platform.naming-strategy=long
            </configurationParameters>
        </properties>
    </configuration>
</plugin>
```            

#### Gradle

```kotlin
tasks.test {
    useJUnitPlatform()
    systemProperty("cucumber.junit-platform.naming-strategy", "long")
}
```

### Use the JUnit Console Launcher ###

* integrate the JUnit Platform Console Launcher | your build

#### Use the Maven Antrun plugin  ####

* | your `pom.xml`

```xml
<dependencies>
   ....
   <dependency>
      <groupId>org.junit.platform</groupId>
      <artifactId>junit-platform-console</artifactId>
      <version>${junit-platform.version}</version>
      <scope>test</scope>
   </dependency>
</dependencies>

<build>
<plugins>
   <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-antrun-plugin</artifactId>
      <executions>
         <execution>
            <!--Work around. Surefire does not use JUnit's Test Engine discovery functionality -->
            <id>CLI-test</id>
            <phase>integration-test</phase>
            <goals>
               <goal>run</goal>
            </goals>
            <configuration>
               <target>
                  <echo message="Running JUnit Platform CLI"/>
                  <java classname="org.junit.platform.console.ConsoleLauncher"
                        fork="true"
                        failonerror="true"
                        newenvironment="true"
                        maxmemory="512m"
                        classpathref="maven.test.classpath">
                     <arg value="--include-engine"/>
                     <arg value="cucumber"/>
                     <arg value="--scan-classpath"/>
                     <arg value="${project.build.testOutputDirectory}"/>
                  </java>
               </target>
            </configuration>
         </execution>
      </executions>
   </plugin>
</plugins>
</build>
```
#### Use the Gradle JavaExec task  ####

* | your `build.gradle.kts`

```kotlin
tasks {

	val consoleLauncherTest by registering(JavaExec::class) {
		dependsOn(testClasses)
		val reportsDir = file("$buildDir/test-results")
		outputs.dir(reportsDir)
		classpath = sourceSets["test"].runtimeClasspath
		main = "org.junit.platform.console.ConsoleLauncher"
		args("--scan-classpath")
		args("--include-engine", "cucumber")
		args("--reports-dir", reportsDir)
	}

	test {
		dependsOn(consoleLauncherTest)
		exclude("**/*")
	}
}
```

### Running a single scenario or feature from the CLI

* `cucumber.features` property
  * Cucumber ignore any other selectors -- from -- JUnit
  * recommendation
    * 👁️execute ONLY the Cucumber engine 👁️

#### Maven

* _Example:_ select `example.feature` | line 10

```shell
mvn test -Dsurefire.includeJUnit5Engines=cucumber -Dcucumber.plugin=pretty -Dcucumber.features=path/to/example.feature:10 
// -Dsurefire.includeJUnit5Engines=cucumber   == ONLY cucumber engine
```

#### Gradle

TODO: (I don't know how. Feel free to send a pull request. ;))

## Suites with different configurations

* == run Cucumber multiple times / different configurations
* steps
  * add

      ```xml
      <dependency>
         <groupId>org.junit.platform</groupId>
         <artifactId>junit-platform-suite</artifactId>
         <version>${junit-platform.version}</version>
         <scope>test</scope>
      </dependency>
      ```

  * define suites / -- use the annotation from -- [`org.junit.platform.suite.api`](https://junit.org/junit5/docs/current/api/org.junit.platform.suite.api/org/junit/platform/suite/api/package-summary.html)


```java
package com.example;

import org.junit.platform.suite.api.ConfigurationParameter;
import org.junit.platform.suite.api.IncludeEngines;
import org.junit.platform.suite.api.SelectPackages;
import org.junit.platform.suite.api.Suite;

import static io.cucumber.junit.platform.engine.Constants.GLUE_PROPERTY_NAME;

@Suite
@IncludeEngines("cucumber")
@SelectPackages("com.example")
@ConfigurationParameter(key = GLUE_PROPERTY_NAME, value = "com.example")
public class RunCucumberTest {
}
```

## Parallel execution 

* By default, Cucumber runs tests sequentially | 1! thread
* Running tests in parallel
  * set `cucumber.execution.parallel.enabled=true` 
    * [ways in Junit5](https://junit.org/junit5/docs/current/user-guide/#running-tests-config-params)
  -- _Example:_ | `junit-platform.properties` --
  * Cucumber supports JUnit 5s `ParallelExecutionConfigurationStrategy`
    * allows controlling
      * desired parallelism
      * maximum parallelism
    * Cucumber built-in implementations
      * `dynamic`
        * `cucumber.execution.parallel.config.strategy=dynamic`
        * desired parallelism = `availableCores` * `cucumber.execution.parallel.config.dynamic.factor`
        * 👁️ default one / `cucumber.execution.parallel.config.dynamic.factor=1` 👁️
      * `fixed`
        * `cucumber.execution.parallel.config.strategy=fixed`
        * `cucumber.execution.parallel.config.fixed.parallelism=desiredParallelism`
        * `cucumber.execution.parallel.config.fixed.max-pool-size=forkJoinPool'sMaximumPoolSize` 
      * `custom`
        * `cucumber.execution.parallel.config.strategy=custom`
        * specify -- via -- `cucumber.execution.parallel.config.custom.class`
  * if `.fixed.max-pool-size` limits the maximum#OfConcurrentThreads -> Cucumber does NOT guarantee that the #OfConcurrentlyExecutingScenarios < `.fixed.max-pool-size`
    * [junit5/#3108](https://github.com/junit-team/junit5/issues/3108)

### Exclusive Resources ###

* 👁️ if you want to avoid flaky tests | multiple scenarios / manipulate SAME resource -> [synchronize the tests][junit5-user-guide-synchronization] 👁️

    [junit5-user-guide-synchronization]: https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution-synchronization

    * if you want to sync a scenario | specific resource -> scenario -- must be -- tagged & tag -- mapped to a -- lock / specific resource
      * resource -- is identified by an -- arbitrary string
        * `@` is added by [junit5/Resources.java][resources-java]
      * ways to lock
        * `read-write-lock`
        * `read-lock`
      * _Example:_ lock the resource `java.lang.System.properties` | 2 scenarios -> NOT concurrently executed both scenarios

```gherkin
Feature: Exclusive resources

   @reads-and-writes-system-properties      // Scenario tagged
   Scenario: first example
      Given this reads and writes system properties
      When it is executed
      Then it will not be executed concurrently with the second example

   @reads-system-properties                 // Scenario tagged
   Scenario: second example
      Given this reads system properties
      When it is executed
      Then it will not be executed concurrently with the first example
```

```properties
cucumber.execution.exclusive-resources.reads-and-writes-system-properties.read-write=java.lang.System.properties
cucumber.execution.exclusive-resources.reads-system-properties.read=java.lang.System.properties
```

[resources-java]: https://github.com/junit-team/junit5/blob/main/junit-jupiter-api/src/main/java/org/junit/jupiter/api/parallel/Resources.java

### Running tests in isolation

* 👁 scenario -- runs while -- NO other scenarios are running 👁️ 
* requirements
  * [`org.junit.platform.engine.support.hierarchical.ExclusiveResource.GLOBAL_KEY`][global-key]

[global-key]: https://github.com/junit-team/junit5/blob/main/junit-platform-engine/src/main/java/org/junit/platform/engine/support/hierarchical/ExclusiveResource.java#L47

* _Example:_

```gherkin
Feature: Isolated scenarios

   @isolated
   Scenario: isolated example
      Given this scenario runs isolated
      When it is executed
      Then it will not be executed concurrently with the second or third example

   Scenario: second example
      When it is executed
      Then it will not be executed concurrently with the isolated example
      And it will be executed concurrently with the third example

   Scenario: third example
      When it is executed
      Then it will not be executed concurrently with the isolated example
      And it will be executed concurrently with the second example
```

```properties
cucumber.execution.exclusive-resources.isolated.read-write=org.junit.platform.engine.support.hierarchical.ExclusiveResource.GLOBAL_KEY
```
### Executing features in parallel 

* By default, if parallel execution is enabled -> scenarios and examples -- are executed -- in parallel 
  * JUnit 4 -- can ONLY execute -- features in parallel
    * Reason: 🧠 limitations 🧠
  * if you want to skip it -> `cucumber.execution.execution-mode.feature=same_thread` 

## Configuration Options ##

* -- inherit from -- JUnit Platform
  * [ways to configure](https://junit.org/junit5/docs/current/user-guide/#running-tests-config-params)
* Cucumber properties
  * check next table & [Constants](src/main/java/io/cucumber/junit/platform/engine/Constants.java)

```
cucumber.ansi-colors.disabled=                                 # true or false. 
                                                               # default: false                     
       
cucumber.filter.name=                                          # a regular expression.
                                                               # only scenarios with matching names are executed.
                                                               # example: ^Hello (World|Cucumber)$
                                                               # note: To ensure consistent reports between Cucumber and
                                                               # JUnit 5 prefer using JUnit 5s discovery request filters
                                                               # or JUnit 5 tag expressions instead.

cucumber.features=                                             # comma separated paths to feature files. 
                                                               # example: path/to/example.feature, path/to/other.feature
                                                               # note: When used any discovery selectors from the JUnit
                                                               # Platform will be ignored. This may lead to multiple
                                                               # executions of Cucumber. For example when used in
                                                               # combination with the JUnit Platform Suite Engine.
                                                               # When using Cucumber through the JUnit Platform
                                                               # Launcher API or the JUnit Platform Suite Engine, it is
                                                               # recommended to use JUnit's DiscoverySelectors or 
                                                               # Junit Platform Suite annotations.

cucumber.filter.tags=                                          # a cucumber tag expression.
                                                               # only scenarios with matching tags are executed.
                                                               # example: @Cucumber and not (@Gherkin or @Zucchini)
                                                               # note: To ensure consistent reports between Cucumber and
                                                               # JUnit 5 prefer using JUnit 5s discovery request filters
                                                               # or JUnit 5 tag expressions instead.

cucumber.glue=                                                 # comma separated package names.
                                                               # example: com.example.glue  

cucumber.junit-platform.naming-strategy=                       # long or short.
                                                               # default: short
                                                               # include parent descriptor name in test descriptor.

cucumber.junit-platform.naming-strategy.short.example-name=    # number or pickle.
                                                               # default: number
                                                               # Use example number or pickle name for examples when
                                                               # short naming strategy is used

cucumber.junit-platform.naming-strategy.long.example-name=     # number or pickle.
                                                               # default: number
                                                               # Use example number or pickle name for examples when
                                                               # long naming strategy is used

cucumber.plugin=                                               # comma separated plugin strings.
                                                               # example: pretty, json:path/to/report.json
 
cucumber.object-factory=                                       # object factory class name.
                                                               # example: com.example.MyObjectFactory

cucumber.publish.enabled                                       # true or false. 
                                                               # default: false
                                                               # enable publishing of test results

cucumber.publish.quiet                                         # true or false. 
                                                               # default: false
                                                               # suppress publish banner after test execution.

cucumber.publish.token                                         # any string value.
                                                               # publish authenticated test results.

cucumber.snippet-type=                                         # underscore or camelcase.
                                                               # default: underscore

cucumber.execution.dry-run=                                    # true or false.
                                                               # default: false

cucumber.execution.execution-mode.feature=                     # same_thread or concurrent
                                                               # default: concurrent
                                                               # same_thread - executes scenarios sequentially in the 
                                                               # same thread as the parent feature 
                                                               # concurrent - executes scenarios concurrently on any
                                                               # available thread

cucumber.execution.parallel.enabled=                           # true or false.
                                                               # default: false

cucumber.execution.parallel.config.strategy=                   # dynamic, fixed or custom.
                                                               # default: dynamic

cucumber.execution.parallel.config.fixed.parallelism=          # positive integer.
                                                               # example: 4

cucumber.execution.parallel.config.fixed.max-pool-size=        # positive integer.
                                                               # example: 4

cucumber.execution.parallel.config.dynamic.factor=             # positive double.
                                                               # default: 1.0

cucumber.execution.parallel.config.custom.class=               # class name.
                                                               # example: com.example.MyCustomParallelStrategy

cucumber.execution.exclusive-resources.<tag-name>.read-write=  # a comma separated list of strings
                                                               # example: resource-a, resource-b.

cucumber.execution.exclusive-resources.<tag-name>.read=        # a comma separated list of strings
                                                               # example: resource-a, resource-b
```

## Supported Discovery Selectors and Filters ## 

* TODO:
JUnit 5 [introduced a test discovery mechanism](https://junit.org/junit5/docs/current/user-guide/#launcher-api-discovery)
as a dedicated feature of the platform itself. This allows IDEs and build tools
to identify tests. Supported `DiscoverySelector`s are:

* `ClasspathRootSelector`
* `ClasspathResourceSelector`
* `ClassSelector`
* `PackageSelector`
* `FileSelector`
* `DirectorySelector`
* `UriSelector`
* `UniqueIdSelector`

The only supported `DiscoveryFilter` is the `PackageNameFilter` and only when
features are selected from the classpath.

### Selecting individual scenarios, rules and examples ###

The `FileSelector` and `ClasspathResourceSelector` support a `FilePosition`.

 * `DiscoverySelectors.selectClasspathResource("rule.feature", FilePosition.from(5))`
 * `DiscoverySelectors.selectFile("rule.feature", FilePosition.from(5))`

The `UriSelector` supports URI's with a `line` query parameter:
  - `classpath:/com/example/example.feature?line=20`
  - `file:/path/to/com/example/example.feature?line=20`

Any `TestDescriptor` that matches the line *and* its descendants will be included in the discovery result. For example,
selecting a `Rule` will execute all scenarios contained within the Rule.

## Tags ##

* Cucumber tags -- are mapped to -- JUnit tags
  * `@` NOT part of the JUnit tag 

```gherkin
@Smoke
@Ignore
Scenario: A tagged scenario
  Given I tag a scenario 
  When I select tests with that tag for execution 
  Then my tagged scenario is executed

@Sanity
Scenario: Another tagged scenario
  Given I tag a scenario 
  When I select tests with that tag for execution 
  Then my tagged scenario is executed
```

* if you use Maven -> tags can be provided from the CLI -- via -- the parameters
  * `groups`
  * `excludedGroups`
  * _Example:_

    ```
    mvn verify -DexcludedGroups="Ignore" -Dgroups="Smoke | Sanity"
    ```

* check
  * [JUnit 5 Suite: @Include Tags](https://junit.org/junit5/docs/current/api/org.junit.platform.suite.api/org/junit/platform/suite/api/IncludeTags.html)
  * [JUnit 5 Suite: @Exclude Tags](https://junit.org/junit5/docs/current/api/org.junit.platform.suite.api/org/junit/platform/suite/api/ExcludeTags.html)
  * [JUnit 5 Console Launcher: Options](https://junit.org/junit5/docs/current/user-guide/#running-tests-console-launcher-options)
  * [JUnit 5 Tag Expression](https://junit.org/junit5/docs/current/user-guide/#running-tests-tag-expressions)
  * [Maven: Filtering by Tags](https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit-platform.html)
  * [Gradle: Test Grouping](https://docs.gradle.org/current/userguide/java_testing.html#test_grouping)

### @Disabled

* TODO:
It is possible to recreate JUnit Jupiter's `@Disabled` functionality by
setting the `cucumber.filter.tags=not @Disabled` property<sup>1</sup>. Any scenarios 
tagged with `@Disabled` will be skipped. See [Configuration Options](#configuration-options)
for more information. 

1. Do note that this is a [Cucumber Tag Expression](https://cucumber.io/docs/cucumber/api/#tags) rather than a JUnit5
   tag expression.

## Aborting Tests

Cucumber supports [OpenTest4Js](https://github.com/ota4j-team/opentest4j)
`TestAbortedException`. This makes it possible to use JUnit Jupiter's
`Assumptions` to abort rather than fail a scenario.

```java
package com.example;

import io.cucumber.java.Before;
import org.junit.jupiter.api.Assumptions;

import java.util.List;

public class RpnCalculatorSteps {

   @Before
   public void before() {
       boolean condition = // decide if tests should abort
       Assumptions.assumeTrue(condition, "Condition not met");
   }
}
```

## Rerunning Failed Scenarios ##

When using `cucumber-junit-platform-engine` rerun files are not supported.
Use either Maven, Gradle or the JUnit Platform Launcher API.

### Using Maven

When running Cucumber through the [JUnit Platform Suite Engine](use-the-jUnit-platform-suite-engine)
use [`rerunFailingTestsCount`](https://maven.apache.org/surefire/maven-surefire-plugin/examples/rerun-failing-tests.html).

Note: any files written by Cucumber will be overwritten during the rerun.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <rerunFailingTestsCount>2</rerunFailingTestsCount>
        <properties>
            <!-- Work around. Surefire does not include enough
                 information to disambiguate between different
                 examples and scenarios. -->
            <configurationParameters>
                cucumber.junit-platform.naming-strategy=long
            </configurationParameters>
        </properties>
    </configuration>
</plugin>
```

### Using Gradle.

Gradle support for JUnit 5 is rather limited 
[gradle#4773](https://github.com/gradle/gradle/issues/4773), 
[junit5#2849](https://github.com/junit-team/junit5/issues/2849).
As a workaround you can the [Gradle Cucumber-Companion](https://github.com/gradle/cucumber-companion)
plugin in combination with [Gradle Test Retry](https://github.com/gradle/test-retry-gradle-plugin)
plugin.

Note: any files written by Cucumber will be overwritten while retrying.

### Using the JUnit Platform Launcher API

```java
package com.example;

import org.junit.platform.engine.discovery.DiscoverySelectors;
import org.junit.platform.engine.discovery.UniqueIdSelector;
import org.junit.platform.launcher.Launcher;
import org.junit.platform.launcher.LauncherDiscoveryRequest;
import org.junit.platform.launcher.TestIdentifier;
import org.junit.platform.launcher.core.LauncherFactory;
import org.junit.platform.launcher.listeners.SummaryGeneratingListener;
import org.junit.platform.launcher.listeners.TestExecutionSummary;
import org.junit.platform.launcher.listeners.TestExecutionSummary.Failure;

import java.util.List;
import java.util.stream.Collectors;

import static org.junit.platform.engine.discovery.DiscoverySelectors.selectDirectory;
import static org.junit.platform.launcher.core.LauncherDiscoveryRequestBuilder.request;

public class RunCucumber {

   public static void main(String[] args) {

      LauncherDiscoveryRequest request = request()
              .selectors(
                      selectDirectory("path/to/features")
              )
              .build();

      Launcher launcher = LauncherFactory.create();
      SummaryGeneratingListener listener = new SummaryGeneratingListener();
      launcher.registerTestExecutionListeners(listener);
      launcher.execute(request);

      TestExecutionSummary summary = listener.getSummary();
      // Do something with summary

      List<UniqueIdSelector> failures = summary.getFailures().stream()
              .map(Failure::getTestIdentifier)
              .filter(TestIdentifier::isTest)
              .map(TestIdentifier::getUniqueId)
              .map(DiscoverySelectors::selectUniqueId)
              .collect(Collectors.toList());

      LauncherDiscoveryRequest rerunRequest = request()
              .selectors(failures)
              .build();

      launcher.execute(rerunRequest);

      TestExecutionSummary rerunSummary = listener.getSummary();
      // Do something with rerunSummary
   }

}
```

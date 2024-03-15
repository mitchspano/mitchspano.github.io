---
layout: post
title: "Custom PMD Rules: Part 2 of 2"
date: 2023-04-28 19:43:24 -0500
---

# Automatically Execute, Report on, and Enforce your PMD Rules

In the [last blog post](https://www.mitchspano.com/blog/custom_pmd_rules_part_1)
we discussed how PMD works and gave some instructions on how to
define and compile your own custom rules using Maven. That was a
great exercise, but the act of explicitly invoking a custom PMD
executable jar from the command line is rather clunky:

```
$JAVA_HOME/bin/java -jar target/pmd-rules-1.0.0-jar-with-dependencies.jar \
    -R category/apex/async-apex.xml \
    -d src/main/default/classes/Example.cls \
    -f json
```

How can we make the execution of these rules and the surfacing of
their findings as effortless as possible? In this post we will share
how to compile your custom PMD rules so that they can be registered
with the SFDX Scanner CLI plugin, and how to automatically execute,
report on, and strictly enforce your organization's rules.

## Project, POM, and Jar

Let's take a look at our project's directory structure:

```
.
├── LICENSE
├── README.md
├── config
│   └── project-scratch-def.json
├── force-app
│   └── main
│       └── default
│           └── classes
│               ├── Example.cls
│               └── Example.cls-meta.xml
├── pom.xml
├── sfdx-project.json
└── src
    └── main
        ├── java
        │   └── com
        │       └── mycompany
        │           └── pmd
        │               ├── AvoidFuture.java
        │               └── RequireFinalizer.java
        └── resources
            └── category
                └── apex
                    └── async-apex.xml
```

In this structure, you'll observe two top level directories of
interest; `force-app` which contains all of the Salesforce parts of our project, and `src` which contains
the Java parts of our project. We are going to change the `pom.xml` up a little bit from
the first post.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.mycompany.pmd</groupId>
  <artifactId>pmd-rules</artifactId>
  <version>1.0.0</version>

  <name>pmd-rules</name>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>11</maven.compiler.source>
    <maven.compiler.target>11</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>net.sourceforge.pmd</groupId>
      <artifactId>pmd</artifactId>
      <version>6.55.0</version>
      <type>pom</type>
    </dependency>
    <dependency>
      <groupId>net.sourceforge.pmd</groupId>
      <artifactId>pmd-apex</artifactId>
      <version>6.55.0</version>
    </dependency>
    <dependency>
      <groupId>net.sourceforge.pmd</groupId>
      <artifactId>pmd-apex-jorje</artifactId>
      <version>6.55.0</version>
      <type>pom</type>
    </dependency>
    <dependency>
      <groupId>org.ow2.asm</groupId>
      <artifactId>asm</artifactId>
      <version>9.4</version>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.8.1</version>
        <configuration>
          <release>11</release>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <configuration>
          <includes>
            <include>com/**/*.class</include>
            <include>category/apex/*.xml</include>
          </includes>
          <archive>
            <manifest>
              <addClasspath>true</addClasspath>
            </manifest>
            <addMavenDescriptor>false</addMavenDescriptor>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

This pom acts slightly differently than before. Instead of compiling
everything into one jar file with the
`maven-assembly-plugin`, this uses the
maven-jar-plugin to produce a scoped jar which just contains the
compiled rule itself.

```
$ mvn package
```

Now, the project will have a file called
`target/pmd-rules-1.0.0.jar`. This jar file is not
executable on its own. If you inspect the contents, you will see
that it only includes the compiled classes and XML files for the
specific rules we have written:

```
$ unzip -l target/pmd-rules-1.0.0.jar


Archive:  target/pmd-rules-1.0.0.jar
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  05-29-2023 17:51   META-INF/
      697  05-29-2023 17:51   META-INF/MANIFEST.MF
        0  05-29-2023 17:51   category/
        0  05-29-2023 17:51   category/apex/
     1240  05-29-2023 17:51   category/apex/async-apex.xml
        0  05-29-2023 17:51   com/
        0  05-29-2023 17:51   com/mycompany/
        0  05-29-2023 17:51   com/mycompany/pmd/
     3263  05-29-2023 17:51   com/mycompany/pmd/RequireFinalizer.class
     1059  05-29-2023 17:51   com/mycompany/pmd/AvoidFuture.class
---------                     -------
     6259                     10 files
```

We could explicitly run our rules by calling the PMD executable and specifying the classPath to include this additional jar, but there is an easier way. The [Salesforce Code Analyzer](https://forcedotcom.github.io/sfdx-scanner/) is a plugin for the SFDX CLI which contains PMD and ESLint and has built-in commands for registering custom rules.

```shell
# Install the SFDX CLI
$ npm install sfdx-cli -g

# Install the sfdx-scanner plugin
$ sfdx plugins:install @salesforce/sfdx-scanner

# Register the custom rules
$ sfdx scanner:rule:add -l apex -p target/pmd-rules-1.0.0.jar

# run the scan
$ sfdx scanner:run -t force-app/main/default/classes/Example.cls --json"
```

#### Findings

```json
{
  "status": 0,
  "result": [
    {
      "engine": "pmd",
      "fileName": "/Users/mitchellspano/Documents/myProject/force-app/main/default/classes/Example.cls",
      "violations": [
        {
          "line": "6",
          "column": "3",
          "endLine": "6",
          "endColumn": "9",
          "severity": 3,
          "ruleName": "AvoidFuture",
          "category": "AsynchronousApex",
          "message": "Usage of the `@Future` annotation should be limited. Please consider implementing the `Queueable` interface instead."
        } // other findings also included
      ]
    }
  ],
  "warnings": [
    "We're continually improving Salesforce Code Analyzer. Tell us what you think! Give feedback at https://research.net/r/SalesforceCA"
  ]
}
```

**Awesome!** This allows us to run our PMD rules in addition to all of the base PMD rules which are already defined. However, we are still running this explicitly from the command line which is not ideal. What we want to do is have the system automatically execute these rules for us whenever a pull request is created or updated, and report the findings back to us in a way which makes sense.

## SFDX Scan Pull Request GitHub Action

The [SFDX Scan Pull Request GitHub action](https://github.com/mitchspano/sfdx-scan-pull-request) (written by yours truly) will execute the SFDX Scanner on the contents of a pull request and report the findings formatted as in-line comments or [checks](https://docs.github.com/en/rest/checks).

<div class="slds-align_absolute-center slds-p-around_medium">
    <img src="/images/pmd/Default-Pmd-Finding.png" style="width: 100%; max-width: 1000px" />
</div>

To use this action in your repository, just reference it in your project:

```yml
name: SFDX Scan Pull Request
on:
  pull_request:
    types: [opened, reopened, synchronize]
  workflow_dispatch:
jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Install SFDX CLI and Scanner
        run: |
          npm install sfdx-cli -g
          sfdx plugins:install @salesforce/sfdx-scanner

      - name: Run SFDX Scanner - Report findings as comments
        uses: mitchspano/sfdx-scan-pull-request@v0.1.14
        with:
          report-mode: comments
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The action has an optional input parameter called [`custom-pmd-rules`](https://github.com/mitchspano/sfdx-scan-pull-request#custom-pmd-rules) which will enable you to register custom rules for use within the scan. Simply enter a JSON string to add any custom rules that need to be registered before the scan is executed. Custom rules are identified by the path to their XML/JAR file and their language.

```yml
- name: Run SFDX Scanner - Report findings as comments
  uses: mitchspano/sfdx-scan-pull-request@v0.1.14
  with:
    report-mode: comments
    custom-pmd-rules: '[{ "path": "target/pmd-rules-1.0.0.jar", "language": "apex" }]'
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Now, when the scan is executed, our custom rules will be included in the scan and any findings will be surfaced to the developers during their regular workflow:

![](/images/pmd/Custom-Rule-Finding.png)

This works well for _reporting_ findings, but what if we want to _enforce_ these rules?

## Enforcement Mechanisms

There are two mechanisms within the scan that can help you to identify a violation as a warning or an error:

The first is called the [`severity-threshold`](https://github.com/mitchspano/sfdx-scan-pull-request#severity-threshold). This is an input parameter which defines a threshold for the severity of the finding. Severity 1 is the highest priority and severity 5 is the lowest. If the threshold parameter is defined and the scan produces a finding at the threshold or more severe, then the action will fail.

The second is called [`strictly-enforced-rules`](https://github.com/mitchspano/sfdx-scan-pull-request#strictly-enforced-rules). This is an input parameter which accepts a JSON string which defines the rules which will be strictly enforced regardless of their severity. Enforced rules are identified by their engine, category, and rule name.

```json
[{ "engine": "pmd", "category": "Performance", "rule": "AvoidDebugStatements" }]
```

If an error is identified, the scan's run will fail with the message
**”One or more errors have been identified within the structure of the code that will need to be resolved before continuing”**

![](/images/pmd/PmdError.png)

_Excellent!_ Now we have some really powerful capabilities. We
can define our own rules, automatically execute a scan to detect
violations of the rules, surface the findings in a readable way, and
strictly enforce the rules we care about. Use these tools to
automate a large portion of the code review process and enforce
adherence to your organization's Apex standards.

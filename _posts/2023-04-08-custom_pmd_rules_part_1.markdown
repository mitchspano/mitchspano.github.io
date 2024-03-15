---
layout: post
title: "Custom PMD Rules: Part 1 of 2"
date: 2023-04-08 19:43:24 -0500
---

# Building and Executing Custom PMD Rules

Most of us are familiar with PMD warnings when writing Apex in Visual Studio Code, but PMD can do much more than that. While PMD provides a standardized set of rules for the Apex programming language, these rules may not cover all the specific coding standards and practices followed by your organization. In this two part blog series, we will:

- Take a look at how PMD works
- Learn a bit about Java and Maven
- Write some custom PMD rules
- Add custom rules to VS Code
- Enforce rules using git

---

## What is PMD?

[PMD](https://pmd.github.io/) is the **P**rogramming **M**istakes **D**etector - an open source static code analysis tool which examines the structure of your Apex class files. PMD can help developers identify structural issues within code such as:

- Improper capitalization
- Queries/DML in loops
- Lacking comments
- …

Most Salesforce developers are used to using PMD distributed as a [plugin for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=chuckjonas.apex-pmd).

![VSCode Sample](/images/pmd/PMD-VSCode-Default-Sample.png)

PMD includes multiple rules out of the box. All PMD rules have a predefined priority: 1 is the highest priority (fix this issue now), 5 is the lowest priority (consider changing this). The default rules are helpful, but they are **not an exhaustive list** of all rules you might want to enforce. You'll want to expand beyond the included rules if any of these are true:

- Your organization has a defined style guide
- Certain issues keep popping up during code review
- Code issues not caught by out of the box rules have caused production issues

## Abstract Syntax Trees

PMD works by taking your Apex class files and converting them into an _abstract syntax tree_ (AST). An AST is a syntactic representation of the structure of a program's source code text. Each node of the tree corresponds to a structural element within the source code. Abstract means that not all details are represented - just structural details. PMD uses abstract syntax trees to understand the structure of your Apex class. Nodes represent things like:

- Methods
- Parameters
- Comments
- Annotations
- Inner classes
- …

Here is an example of some Apex and its corresponding AST:

```java
while (b != 0) {
  if (a > b) {
    a = a-b;
  } else {
    b = b-a;
  }
}
return a;
```

![](/images/pmd/AST.png)

Once PMD has generated the AST for a given class file, it uses the [Visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern) to traverse the tree and apply the rules.

---

## Java and Maven Basics

To extend PMD beyond its default capabilities, we will need to review some of the basics of Java and Maven.

### Java

Java is a popular programming language that is widely used for developing desktop, mobile, and web applications. It was first released by Sun Microsystems in 1995 and is now owned by Oracle Corporation. Java is known for its "write once, run anywhere" philosophy, meaning that Java code can be compiled into platform-independent bytecode that can run on any platform with a Java Virtual Machine (JVM). Java was originally created for interactive cable TV boxes, but has since proliferated into almost all industries. The Apex syntax that Salesforce developers are familiar with is based on Java, so some of the source code we will go over in this post should look familiar.

Java source code is saved in files with a **.java** file extension. These files contain the human-readable code that is written by developers. In order to run a Java program, the source code must first be compiled into bytecode using the `javac` command. This command takes the Java source code as input and generates a compiled version of the code in files with a **.class** file extension. The compiled **.class** files can then be executed on any platform with a JVM installed. The JVM interprets the bytecode and executes it on the host system.

Java programs can also be distributed in a **.jar** archive, which is a Java-specific archive format that can contain compiled .class files, as well as other resources such as images, audio files, and configuration files. JAR files are commonly used to distribute Java libraries, which are collections of code that can be reused across multiple projects. JAR files can be executed directly using the java command or can be included as dependencies in other Java projects.

### Maven

Maven is a popular build automation tool for Java projects that helps manage the build process and dependencies of a project. It is based on a project object model or POM file that describes the project's structure, dependencies, and build process. One of the key benefits of Maven is its ability to manage project dependencies through the use of a central repository. The Maven repository is a collection of pre-built libraries and plugins that can be downloaded and used in a project. When a dependency is declared in the POM, Maven automatically downloads the necessary library from the repository and makes it available to the project.

The POM is an XML file that contains information about the project, such as its name, version, and dependencies. It also defines the various build phases and goals that can be executed by Maven. The POM can be used to manage dependencies, plugins, and other project-related configuration.

The `mvn package` command is used to build and package the project into a distributable format, such as a JAR file. This command compiles the source code, runs any tests, and packages the compiled code into an executable format. The resulting package can then be distributed or deployed to a server for use.

---

## Asynchronous Apex

Now that we have familiarized ourselves with Java and Maven, let's take a look at some common programming constructs that we use within Salesforce to accomplish asynchronous execution. Asynchronous Apex is an area where the out of the box PMD rules do not offer much utility, so we will need to expand PMD to ensure we are writing asynchronous Apex properly.

### @Future

The `@future` annotation is a feature within the Apex programming language that allows for asynchronous execution of code. When a method is annotated with @future, it is executed in a separate transaction, allowing the main transaction to continue executing without waiting for the @future method to complete.

The @future annotation has its limits though. Methods annotated with @future must be static and can only accept primitive data types as parameters. Additionally, they have no ability to chain or handle exceptions. While @future was once a useful tool for performing long-running or computationally expensive tasks, the preferred method of handling asynchronous Apex is now by implementing the queueable interface.

### Queueable

The `queueable` interface is a more robust and flexible way of handling asynchronous Apex. It allows for more complex data types to be passed as parameters, supports chaining of jobs, and provides better error handling through the use of exception propagation. Queueable jobs can also be monitored and managed through the Apex flex queue, providing greater visibility and control over asynchronous execution.

To implement the queueable interface, a class must define a `public void execute(QueueableContext qc)` method. This method will contain the code that should be executed asynchronously. A new instance of a queueable class can be created and added to the Apex flex queue using the `System.enqueueJob` method.

### Finalizer

The transaction `finalizer` interface is a relatively new feature of Apex which allows an implementation of the `queueable` interface to define additional code to be executed after a queueable job has completed. This feature is useful in situations where follow-up actions are required after a job has finished, or handling any exceptions which may have interrupted the job's execution. Once defined, the transaction finalizer can be added to the queueable job by calling the `System.attachFinalizer(Finalizer f)` method.

### Asynchronous Rules

There are two custom rules we would like to define regarding how asynchronous Apex should be written:

#### 1) Do not use the `@Future` annotation within a class.

We want to warn the author of Apex whenever they write a method using the `@Future` annotation. We should provide the reader of the finding with instructions to use the `queueable` interface instead.

#### 2) Require the attachment of a `Finalizer` within each `Queueable` class.

We want to warn the author of Apex whenever they write a class implementing the `queueable` interface and they fail to attach a `finalizer`. We should explain that the lack of a `finalizer` provides no handling capabilities should the queueable action fail.

---

## Custom Rules

Alright, now with all of that background knowledge, let's get into the good stuff - how to actually write custom PMD rules. First, let's examine the project's layout.

```
├── README.md
├── config
│   └── project-scratch-def.json
├── category
│   └── apex
│       └── async-apex.xml
├── pom.xml
├── sfdx-project.json
└── src
    └── main
        ├── default
        │   └── classes
        │       ├── Example.cls
        │       └── Example.cls-meta.xml
        └── java
            └── com
                └── mycompany
                    └── pmd
                        ├── AvoidFuture.java
                        └── RequireFinalizer.java
```

You'll see that in this case, we are keeping both our sfdx and Java project together in one repository. This is for demonstration purposes - you might want to separate them in your team's workflow, so keep that in mind.

### AvoidFuture.java

Let's write our first rule. Recall that these are our requirements: **Do not use the @Future annotation within a class.**

```java
/*
   Copyright 2023 Google LLC
   SPDX-License-Identifier: Apache-2.0
*/
package com.mycompany.pmd;

import net.sourceforge.pmd.lang.apex.ast.ASTAnnotation;
import net.sourceforge.pmd.lang.apex.rule.AbstractApexRule;

public class AvoidFuture extends AbstractApexRule {

    private static final String FUTURE = "Future";

    @Override
    public Object visit(ASTAnnotation node, Object data) {
        if (FUTURE.equals(node.getImage())) {
            asCtx(data).addViolation(node);
        }
        return data;
    }
}
```

We can see that this class is pretty simple - when visiting any `ASTAnnotation` node within the AST, it just checks if the annotation equals `@Future`, and if so it will add an error to the node. This is a rather trivial example, so let's move onto something more complex.

### RequireFinalizer.java

Let's write our next rule. Recall that these are our requirements: **Require the attachment of a `Finalizer` within each `Queueable` class.**

```java
/*
   Copyright 2023 Google LLC
   SPDX-License-Identifier: Apache-2.0
*/
package com.mycompany.pmd;

import java.util.List;
import net.sourceforge.pmd.lang.apex.ast.ASTMethod;
import net.sourceforge.pmd.lang.apex.ast.ASTMethodCallExpression;
import net.sourceforge.pmd.lang.apex.ast.ASTUserClass;
import net.sourceforge.pmd.lang.apex.ast.ASTParameter;
import net.sourceforge.pmd.lang.apex.rule.AbstractApexRule;

public class RequireFinalizer extends AbstractApexRule {

    private static final String EXECUTE = "execute";
    private static final String QUEUEABLE = "queueable";
    private static final String QUEUEABLE_CONTEXT = "queueablecontext";
    private static final String SYSTEM_ATTACH_FINALIZER = "system.attachfinalizer";

    @Override
    public Object visit(ASTUserClass topLevelClass, Object data) {
        scanClassForViolation(topLevelClass, data);
        for (ASTUserClass innerClass : topLevelClass.findDescendantsOfType(ASTUserClass.class)) {
            scanClassForViolation(innerClass, data);
        }
        return data;
    }

    private void scanClassForViolation(ASTUserClass theClass, Object data) {
        if (!implementsTheQueueableInterface(theClass)) {
            return;
        }
        for (ASTMethod theMethod : theClass.findDescendantsOfType(ASTMethod.class)) {
            if (isTheExecuteMethodOfTheQueueableInterface(theMethod)
                    && !callsTheSystemAttachFinalizerMethod(theMethod)) {
                asCtx(data).addViolation(theMethod);
            }
        }
    }

    private boolean implementsTheQueueableInterface(ASTUserClass theClass) {
        for (String interfaceName : theClass.getInterfaceNames()) {
            if (interfaceName.equalsIgnoreCase(QUEUEABLE)) {
                return true;
            }
        }
        return false;
    }

    private boolean isTheExecuteMethodOfTheQueueableInterface(ASTMethod theMethod) {
        if (!theMethod.getCanonicalName().equalsIgnoreCase(EXECUTE)) {
            return false;
        }
        List<ASTParameter> parameters = theMethod.findDescendantsOfType(ASTParameter.class);
        return parameters.size() == 1 && parameters.get(0).getType().equalsIgnoreCase(QUEUEABLE_CONTEXT);
    }

    private boolean callsTheSystemAttachFinalizerMethod(ASTMethod theMethod) {
        for (ASTMethodCallExpression methodCallExpression : theMethod
                .findDescendantsOfType(ASTMethodCallExpression.class)) {
            if (methodCallExpression.getFullMethodName().equalsIgnoreCase(SYSTEM_ATTACH_FINALIZER)) {
                return true;
            }
        }
        return false;
    }
}
```

This rule is slightly more complex. The `visit` method is overridden and defined for visiting an `ASTUserClass` instead of `ASTAnnotation` node like the previous example. It will scan the top level class and any inner classes to see if they implement the `queueable` interface. If they do, the code will scan all methods to find the `public void execute(QueueableContext qc)` method and ensure that the `System.attachFinalizer(Finalizer f)` method is called within the `execute` method. If no finalizer is attached, an error will be added to the `execute` method.

### Ruleset

Now that we have the source code written for our rules, we can identify if and where we need to add an error when a class is scanned using PMD. However, you may have noticed that we did not define the error's priority or error message - that's where the `acync-apex.xml` ruleset file comes into play. Ruleset files within PMD allow us to assign a name, priority, message, and category to our compiled rules. Here are the contents of `acync-apex.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ruleset xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    name="AsynchronousApex"
    xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 https://pmd.sourceforge.io/ruleset_2_0_0.xsd">
    <description>Asynchronous Apex</description>
    <rule
        name="AvoidFuture"
        language="apex"
        class="com.mycompany.pmd.AvoidFuture"
        message="Usage of the `@Future` annotation should be limited. Please consider implementing the `Queueable` interface instead."
    >
        <description>Avoid @Future annotation - recommend using queueable instead</description>
        <priority>3</priority>
    </rule>
    <rule
        name="RequireFinalizer"
        language="apex"
        class="com.mycompany.pmd.RequireFinalizer"
        message="It is best practice to call the `System.attachFinalizer(Finalizer f)` method within the `execute` method of a class which implements the `Queueable` interface. This will enable the handling of unsuccessful asynchronous processing."
    >
        <description>Each Queueable class should attach a finalizer</description>
        <priority>3</priority>
    </rule>
</ruleset>
```

---

## Compilation and Execution

Now that we have our rules written and our ruleset file defined, we will use Maven to compile and build an executable program which contains our rules.

### POM

The POM file is key to the build step. It will enable us to get the dependencies required for PMD and construct an executable `.jar` file.

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
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <version>3.3.0</version>
        <configuration>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
          <archive>
            <manifest>
              <mainClass>net.sourceforge.pmd.PMD</mainClass>
            </manifest>
          </archive>
        </configuration>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

When the `mvn package` command is executed with this POM file, Maven will download the dependencies listed in the dependencies section. Maven will then compile the source code and create a JAR file with dependencies included using the `maven-assembly-plugin`. The JAR file will be named `pmd-rules-1.0.0-jar-with-dependencies.jar` and will contain the compiled Java classes, along with all the necessary dependencies packaged inside. The main class of the JAR file will be `net.sourceforge.pmd.PMD`, which is specified in the `mainClass` element of the `maven-assembly-plugin` configuration. Finally, the JAR file will be placed in the `target` directory of the project.

### Execute

Now, we have an executable program within the `pmd-rules-1.0.0-jar-with-dependencies.jar` file. Let's examine the `Example.cls` before we execute the scan:

```java
/*
   Copyright 2023 Google LLC
   SPDX-License-Identifier: Apache-2.0
*/

public class Example {

  @Future
  public static void deactivateUsers(Set<Id> userIds) {
    List<User> usersToUpdate = new List<User>();
    for (Id i : userIds) {
      usersToUpdate.add(new User(Id = i, IsActive = false));
    }
    update usersToUpdate;
  }

  public class UserUpdater implements Queueable, Finalizer {
    private List<User> usersToUpdate;
    public UserUpdater(List<User> usersToUpdate) {
      this.usersToUpdate = usersToUpdate;
    }

    public void execute(QueueableContext context) {
      update usersToUpdate;
    }

    public void execute(FinalizerContext ctx) {
      if (ctx.getResult() == ParentJobResult.SUCCESS) {
        // Handle success
      } else {
        // Handle failure
      }
    }
  }
}
```

You'll see that the class has an `@Future` annotated method, and it contains an inner `queueable` which also implements `finalizer`, but does not attach the finalizer. Now, let's execute our custom executable version of PMD and see our new rules in action.

```
$JAVA_HOME/bin/java -jar target/pmd-rules-1.0.0-jar-with-dependencies.jar \
    -R category/apex/async-apex.xml \
    -d src/main/default/classes/Example.cls \
    -f json
```

This will produce the following output:

```json
{
  "formatVersion": 0,
  "pmdVersion": "6.55.0",
  "timestamp": "2023-01-01T00:00:00.000-00:00",
  "files": [
    {
      "filename": "src/main/default/classes/Example.cls",
      "violations": [
        {
          "beginline": 8,
          "begincolumn": 3,
          "endline": 8,
          "endcolumn": 9,
          "description": "Usage of the `@Future` annotation should be limited. Please consider implementing the `Queueable` interface instead.",
          "rule": "AvoidFuture",
          "ruleset": "AsynchronousApex",
          "priority": 3
        },
        {
          "beginline": 23,
          "begincolumn": 17,
          "endline": 25,
          "endcolumn": 5,
          "description": "It is best practice to call the `System.attachFinalizer(Finalizer f)` method within the `execute` method of a class which implements the `Queueable` interface. This will enable the handling of unsuccessful asynchronous processing.",
          "rule": "RequireFinalizer",
          "ruleset": "AsynchronousApex",
          "priority": 3
        }
      ]
    }
  ],
  "suppressedViolations": [],
  "processingErrors": [],
  "configurationErrors": []
}
```

Excellent! Our rules are working as we intend and finding the issues we expect. Let's modify our class to resolve these issues and run the scan again:

```java
/*
   Copyright 2023 Google LLC
   SPDX-License-Identifier: Apache-2.0
*/

public class Example {

  public class UserUpdater implements Queueable, Finalizer {
    private List<User> usersToUpdate;
    public UserUpdater(List<User> usersToUpdate) {
      this.usersToUpdate = usersToUpdate;
    }

    public void execute(QueueableContext context) {
      System.attachFinalizer(this);
      update usersToUpdate;
    }

    public void execute(FinalizerContext ctx) {
      if (ctx.getResult() == ParentJobResult.SUCCESS) {
        // Handle success
      } else {
        // Handle failure
      }
    }
  }
}
```

```
$JAVA_HOME/bin/java -jar target/pmd-rules-1.0.0-jar-with-dependencies.jar \
    -R category/apex/async-apex.xml \
    -d src/main/default/classes/Example.cls \
    -f json
```

```json
{
  "formatVersion": 0,
  "pmdVersion": "6.55.0",
  "timestamp": "2023-01-01T00:00:00.000-00:00",
  "files": [],
  "suppressedViolations": [],
  "processingErrors": [],
  "configurationErrors": []
}
```

Awesome! Our modifications to fix the identified findings have been
made and PMD no longer finds any issues with our code &#x1f973;.

You can see that writing custom PMD rules is not that difficult once
you understand all of the moving parts. You can use PMD to write,
compile, and execute custom rules that match your organization's
coding standards. Running these rules from the command line is a
bit... _clunky_ though. In the next post, we will discuss how
to make violations of these rules visible within VS Code and how to
programmatically enforce these rules to ensure that they are never
violated within your code base.

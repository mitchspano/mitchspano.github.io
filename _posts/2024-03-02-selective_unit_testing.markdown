---
layout: post
title: "Deploy Faster, Test Smarter: The Selective Unit Testing Strategy Every Salesforce Org Needs"
date: 2024-03-09 19:43:24 -0500
---

# How Selective Unit Testing Can Improve Your Salesforce Development Cycle

Developing in Salesforce can be a breeze, but unit testing your code against the
entire org can quickly turn into a waiting game.

\
With Salesforce's org-based development model,
running all unit tests during development can lead to excruciatingly long execution times,
hindering your development flow and slowing down your ability to iterate quickly.

\
This post outlines a selective unit testing strategy that
allows you to focus on the most relevant tests during pull request creation. By strategically
targeting specific code changes, you'll ensure high-quality code while reclaiming precious time
in your development cycle.

---

## An Imperfect Solution

In a perfect world, I would advocate for running every single unit test during development.
After all, comprehensive testing is the cornerstone of building robust software.
Extensive testing helps to uncover hidden bugs early in the development lifecycle, preventing
them from reaching production and potentially impacting real users.

\
Unfortunately, the dream of running all unit tests quickly in Salesforce development has historically
been hampered by a key factor: tightly coupled tests. Traditionally, Salesforce developers haven't
always isolated their unit tests from the org's data, meaning tests rely on interacting with the
actual Salesforce database. This heavy reliance significantly inflates test execution times,
sometimes ballooning to multiple hours for a comprehensive suite.

\
While there are published strategies, like those championed in this very blog (see
[Pure Unit Testing in Apex](https://www.mitchspano.com/blog/pure_unit_testing_in_apex)),
to write 'pure unit tests' that minimize database interaction and improve performance,
these practices haven't been universally adopted. The reality is, many developers continue
to write tests that heavily interact with the org, leading to the de-facto state of agonizingly
slow execution times. This is where selective unit testing comes in - it empowers developers
to focus their efforts on a strategically chosen subset of tests, ensuring high code quality
within the constraints of current development practices.

---

## Applicable for any Git Provider

While the specifics of this blog post will be geared towards developers using GitHub for pull
requests, the core principles of selective unit testing are readily applicable to any Git
provider, including Bitbucket, GitLab, or your preferred platform. These platforms all offer
mechanisms to trigger workflows upon pull request creation, allowing you to leverage selective
unit testing strategies regardless of your chosen version control system.

---

## Moving Parts

Now that we've established the 'why' behind selective unit testing in Salesforce, let's dive
into the 'how.' This section explores three key strategies to help you target the most relevant
unit tests for each pull request. We'll delve into mechanisms for:

- **Automatic Targeting**: Automatically executing any unit test that falls within the scope
  of the pull request's changes.
- **Comment-Driven Selection**: Specifying an enumerated list of tests to be executed directly
  within the pull request comment body.
- **Always-On Testing**: Defining a core set of tests that will always run, regardless of
  the specific pull request content.

The general approach is to use these three strategies to collect a set of tests to be executed,
then use those tests in your `sf project deploy validate` command to run these tests.

---

## Automatic Targeting

This is the most straightforward of the strategies; we just want to identify all test classes
which exist within the pull request.

```js
/*
  Copyright 2023 Google LLC
  SPDX-License-Identifier: Apache-2.0
*/
function getTestsInPullRequest(fromCommit, toCommit) {
  if (!fromCommit || !toCommit) {
    console.error(
      "Error: Please provide both 'fromCommit' and 'toCommit' SHAs as arguments."
    );
    process.exit(1);
  }
  return execSync(
    `git diff --name-only --diff-filter=d "${fromCommit}"..."${toCommit}" "*.cls"`
  )
    .toString()
    .trim()
    .split("\n")
    .filter((file) => {
      return execSync(`git show ${toCommit}:${file}`)
        .toString()
        .trim()
        .toLowerCase()
        .includes("@istest");
    })
    .map((filePath) => path.basename(filePath, ".cls"));
}
```

## Comment-Driven Selection

The next strategy is to allow developers and administrators to enumerate which specific
tests they would like to execute with this specific pull request.
\
To support this, we enable teammates to include a formatted text within the pull request's
description.

```
TESTS=[YourTest, SomeOtherTest, YetAnotherTest]
```

We can extract this from a sanitized version of the pull request's description.
\
This is particularly helpful in the situation where you modify an existing
class without modifying its existing test class; just list the test class
in the pull request's description and it will be executed.

```js
/*
  Copyright 2023 Google LLC
  SPDX-License-Identifier: Apache-2.0
*/
function getTestNamesFromDescription(description) {
  if (!description) {
    return [];
  }
  const testMatch = description.match(/TESTS=[([^]]*)]/); // Regex for TESTS=[] format
  if (!testMatch) {
    return [];
  }
  return testMatch[1].split(",").map((test) => test.trim());
}
```

## Always-On Testing

The next strategy is to allow developers and administrators to enumerate which specific
tests that should always run in a file in the project's directory structure.
This is helpful in the situation where you have tests which could break without code modification.
For example, if you have unit tests which verify validation rules within the application.
If someone were to raise a pull request which just modified a validation rule, the tests
included in the file can validate that the rule is still working properly.

\
To support this, teams can just specify a file such as `.alwaysRun` with line delimited tests:

```
# .alwaysRun
MyTest
YourTest
```

Once this is defined, we can simply read the file based on its path and include these tests:

```js
/*
  Copyright 2023 Google LLC
  SPDX-License-Identifier: Apache-2.0
*/
function readFileLines(filePath) {
  if (!filePath) {
    return [];
  }
  try {
    const fileContents = fs.readFileSync(filePath, "utf8");
    return fileContents
      .trim()
      .split("\n")
      .map((line) => line.trim());
  } catch (error) {
    console.error(`Error reading file: ${error.message}`);
    return [];
  }
}
```

## Putting it all Together

Now we can put it all together and generate the set of tests to run in a GitHub action on the
pull request.

```js
/*
  Copyright 2023 Google LLC
  SPDX-License-Identifier: Apache-2.0
*/
const { execSync } = require("child_process");
const path = require("path");
const fs = require("fs");

function getTestsInPullRequest(fromCommit, toCommit) {
  // ...
}

function getTestNamesFromDescription(description) {
  // ...
}

function readFileLines(filePath) {
  // ...
}

function flattenAndDeduplicate(array1, array2, array3) {
  const allItems = [...array1, ...array2, ...array3];
  return [...new Set(allItems)];
}

function main() {
  const fromCommit = process.argv[2];
  const toCommit = process.argv[3];
  const description = process.argv[4];
  const alwaysRunFile = process.argv[5];

  const allTests = flattenAndDeduplicate(
    getTestsInPullRequest(fromCommit, toCommit),
    getTestNamesFromDescription(description),
    readFileLines(alwaysRunFile)
  );
  console.log(allTests.join(","));
}

main();
```

![](/images/selectiveUnitTesting/pr_with_tests_specified.png)

![](/images/selectiveUnitTesting/tests_in_pr.png)

```yml
name: Demo
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
      - name: Selective Test Generation
        run: |
          # Sanitize the input from the pull request's description
          pr_description=$(cat << EOF
          ${{ github.event.pull_request.body }}
          EOF)
          node selectiveTests.cjs "HEAD^" "HEAD" "${pr_description}" ".alwaysRun"
```

![](/images/selectiveUnitTesting/action_run.png)

## Conclusion

By leveraging a combination of Automatic Targeting, Comment-Driven Selection, and Always-On Testing,
Salesforce development teams can achieve a robust and efficient selective test execution strategy.
This approach streamlines the testing process by focusing efforts on the most relevant test suites,
significantly reducing execution times compared to running all tests for every pull request. This improves
development velocity by removing the pain of slow test cycles - making the best of the less than
ideal situation most Salesforce orgs find themselves in.

\
By embracing these mechanisms, teams can confidently deliver high-quality code while maintaining
a streamlined development workflow.

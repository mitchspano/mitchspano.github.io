---
layout: post
mathjax: true
title: "There Can Be Only One (Branch): Trunk-Based Development for Salesforce"
date: 2025-02-01 01:43:24 -0500
---

## Trunk Based Development

In the Salesforce development world, a common practice has been to rely on
persistent, long-lived branches for non-production environments like QA and UAT
While seemingly straightforward, this branching strategy often introduces a host
of challenges that can significantly hinder development velocity and release
cadence. From the ever-present threat of merge conflicts and the insidious creep
of environment drift, to the complexities of deployment and the frustrating
repetition of steps within the development pipeline, these persistent branches
can become a major bottleneck for your Salesforce development team. This post
explores the pitfalls of this traditional branching approach within Salesforce
and introduces a more streamlined and efficient alternative: Trunk-Based
Development.

### The Status Quo

Many Salesforce development teams employ a branching strategy where each
persistent branch corresponds directly to a specific environment. This is a
common system landscape you're likely to encounter, though the specific names
and number of branches/environments may vary. For instance, you might see
branches labeled `QA`, `UAT`, and `main`, mirroring corresponding sandboxes and
the production org. In this topology, `QA` serves as the source of truth for the
QA sandbox, `UAT` for the UAT sandbox, and `main` for production. This setup,
while prevalent, often introduces challenges related to merge conflicts,
environment drift, and deployment complexity.

The process of promoting changes up the environments involves merging feature
branches into `QA`. Critically, when a pull request (PR) containing these
changes is created or updated, the scope of the changes is validated against the
target Salesforce environment (in this case, the QA sandbox). This validation
typically includes performing static code analysis, validation of the
deployment, running Apex tests, and checking for code coverage. Once the PR is
approved and merged into the `QA` branch, the changes are then deployed to the
QA sandbox. From there, changes are often cherry-picked (or sometimes merged)
across the branches – from `QA` to `UAT`, and finally from `UAT` to `main`. Each
promotion up the chain initiates its own validation and deployment in the
corresponding environment and the processes is repeated n times for n
environments.

The `QA`, `UAT`, and `main` branches are not ephemeral; they are
ever-persistent, extending throughout the project's entire lifespan. This
longevity creates several problems and the multi-step process, while seemingly
controlled, introduces significant overhead and risk, as we'll explore further
in this post.

<!-- prettier-ignore-start -->
<div class="plantuml-container">
<!-- prettier-ignore-end -->

<!-- prettier-ignore-start -->
{% plantuml %}
@startuml

scale 1.5
skinparam sequenceArrowThickness 2.5

skinparam cloud<<cloudStyle>> {
  BackgroundColor #3baed3
  FontColor #ffffff
}

skinparam card<<branch>> {
  BackgroundColor #FF0000
  FontColor #ffffff
}


hide stereotype

folder "Developer's Responsibility" {
  cloud Scratch <<cloudStyle>>

  card Feature_branch as "<&fork> **feature/xyz**" <<branch>>
  card QA_branch as "<&fork> **QA**" <<branch>>
  card UAT_branch as "<&fork> **UAT**" <<branch>>
  card main_branch as "<&fork> **main**" <<branch>>

  Scratch -[#black,dashed]d-> Feature_branch : " commit"
  Feature_branch -[#black]r-> QA_branch : "merge"
  QA_branch -[#black]r-> UAT_branch : "🍒⛏️"
  UAT_branch -[#black]r-> main_branch : "🍒⛏️"
}

folder "System's Responsibility" {
  cloud QA <<cloudStyle>>
  cloud UAT <<cloudStyle>>
  cloud Production <<cloudStyle>>
}

QA_branch -[#orange]d-> QA
QA_branch -[#green]d-> QA

UAT_branch -[#orange]d-> UAT
UAT_branch -[#green]d-> UAT

main_branch -[#orange]d-> Production
main_branch -[#green]d-> Production

legend
  | Arrow | Description |
  | <color:black><size:18><&arrow-right></size></color>   | Pull request / cherry-pick |
  | <color:orange><size:18><&arrow-right></size></color>  | Validation |
  | <color:green><size:18><&arrow-right></size></color>   | Deployment |
endlegend

@enduml
{% endplantuml %}
<!-- prettier-ignore-end -->

<!-- prettier-ignore-start -->
</div>

<!-- prettier-ignore-end -->

In the traditional paradigm, the developer is responsible for managing the
promotion of components repeatedly across multiple branches. The only thing that
the system automatically performs is the validation and deployment to the
respective Salesforce environments.

### Issues with the Status Quo

While the described branching strategy might appear organized at first glance,
it introduces several significant issues that can severely hamper a Salesforce
development team's efficiency and reliability.

#### Repeated Steps

One of the most glaring problems is the constant repetition of steps. Promoting
changes from development to QA, then to UAT, and finally to production requires
developers to perform the same actions multiple times. They must manually invoke
the cherry-pick process for every change moving between branches. This manual
intervention is not only time-consuming but also prone to human error.

#### Environment Drift

This manual cherry-picking process also leaves the project ripe for creating
drift between environments. Because changes are manually selected and applied to
each branch, it's easy to accidentally omit a crucial component or introduce
inconsistencies. Over time, these small discrepancies accumulate, leading to
significant differences between the QA, UAT, and production environments.
Reconciling this drift becomes a major undertaking, often requiring extensive
debugging and rework, particularly when trying to diagnose production issues
that can't be easily reproduced in lower environments.

#### Manual Selection and Promotion

Finally, all of these challenges are amplified substantially when multiple
related components need to be promoted together from one branch to another.
Imagine a scenario where a feature spans several Apex classes, Visualforce
pages, and other metadata types. Accurately assembling the exact set of files
necessary to facilitate a clean and consistent deployment becomes extremely
challenging. Developers must meticulously track each individual change and
ensure it's included in the cherry-pick or merge operation. This complexity
increases the risk of deploying incomplete or inconsistent changes, further
exacerbating the problem of environment drift and increasing the likelihood of
deployment failures.

## What is Trunk-Based Development?

Trunk-Based Development presents a significantly different approach to managing
source code and deployments, offering numerous advantages over the traditional
branching model. The core principle is simple: teams work directly on a single
branch, commonly referred to as `main` or the "trunk." Instead of long-lived
feature branches that diverge significantly from the main codebase, developers
create short-lived feature branches, typically lasting no more than a few days.
These branches are created directly from `main`, and once the feature is
complete and tested, they are merged directly back into `main`. This frequent
integration ensures that the main branch remains stable and always reflects the
latest state of the project.

Releases in Trunk-Based Development are handled differently as well. Instead of
releasing whenever changes are merged into `main`, a specific historical commit
on `main` is chosen as the release point at a scheduled cadence. This approach
greatly simplifies the release process, eliminating the need for complex branch
management and cherry-picking. This is illustrated in the diagram below:

<!-- prettier-ignore-start -->
<div class="plantuml-container">
<!-- prettier-ignore-end -->

<!-- prettier-ignore-start -->
{% plantuml %}
@startuml

scale 1.5
skinparam sequenceArrowThickness 3.5

!define DARK_BLUE #286090
!define LIGHT_BLUE #62b3ce
!define GREEN #157215

together main {
  circle start as "main"
  circle c1 as " "
  circle c2 as " "
  circle c3 as " "
  circle c4 as " "
  circle c5 as " "
  circle c6 as " "
  circle c7 as " "
  circle c8 as " "
  circle c9 as " "
  circle end as "HEAD" 
}

together Features {
  circle f1 as "<&fork> feature_abc"
  circle f2 as " "
  circle f3 as " "
  circle f4 as "<&fork> feature_def"
  circle f5 as " "
  circle f6 as " "
  circle f7 as "<&fork> feature_xyz"
}

together Releases {
  card r1 as "Release 8.1"
  card r2 as "Release 8.2"
}

start -[DARK_BLUE]r-> c1
c1 -[DARK_BLUE]r-> c2
c2 -[DARK_BLUE]r-> c3
c3 -[DARK_BLUE]r-> c4
c4 -[DARK_BLUE]r-> c5
c5 -[DARK_BLUE]r-> c6
c6 -[DARK_BLUE]r-> c7
c7 -[DARK_BLUE]r-> c8
c8 -[DARK_BLUE]r-> c9
c9 -[DARK_BLUE]r-> end

c1 -[LIGHT_BLUE]d-> f1
f1 -[LIGHT_BLUE]r-> f2
f2 -[LIGHT_BLUE]r-> c3

c4 -[LIGHT_BLUE]d-> f3
f3 -[LIGHT_BLUE]r-> f4
f4 -[LIGHT_BLUE]r-> f5
f5 -[LIGHT_BLUE]r-> c6

c7 -[LIGHT_BLUE]d-> f6
f6 -[LIGHT_BLUE]r-> f7
f7 -[LIGHT_BLUE]r-> c9

c4 -[GREEN]u-> r1
c7 -[GREEN]u-> r2

Releases -[hidden]d- main
main -[hidden]d- Features

legend
  | Arrow | Description |
  | <color:DARK_BLUE><size:18><&arrow-right></size></color>   | main |
  | <color:LIGHT_BLUE><size:18><&arrow-right></size></color>  | feature |
  | <color:GREEN><size:18><&arrow-right></size></color>   | release |
endlegend

@enduml
{% endplantuml %}
<!-- prettier-ignore-end -->

<!-- prettier-ignore-start -->
</div>

<!-- prettier-ignore-end -->

- **Short-lived feature branches:** The light blue lines represent short-lived
  feature branches branching off and merging back into `main`.
- **Direct merge to main:** Feature branches are merged directly back into
  `main` (represented by the light blue lines returning to the dark blue `main`
  line).
- **Release from main:** Releases are created from specific points (commits) on
  `main` (represented by the green lines pointing to the release cards).

### Quality

In addition to reduced steps and simplified deployments, a natural consequence
of trunk-based development practices is often higher quality software. This
improvement stems from a shift in developer mindset and increased ownership.

When developers know that the contents of their pull request (PR) will soon be
deployed to production, they naturally tend to make higher-quality changes. The
immediacy of the deployment creates a stronger sense of responsibility for the
code they are committing. Knowing that their work will be live and potentially
impacting users shortly after it's merged encourages developers to be more
thorough in their testing and more attentive to detail. They are less likely to
cut corners or push code that they aren't confident in.

This "production proximity" effect is a powerful motivator for quality.
Developers are more likely to:

- **Write better tests:** Knowing that their code will be in production soon
  incentivizes developers to write more comprehensive unit and integration
  tests. They want to be confident that their changes won't break existing
  functionality or introduce new bugs.

- **Perform more thorough code reviews:** The emphasis on merging frequently and
  deploying rapidly encourages more focused and effective code reviews.
  Reviewers are more likely to scrutinize changes when they know they will be
  live soon.

- **Pay closer attention to edge cases:** Developers are more likely to consider
  edge cases and potential problems when they know their code is going to
  production quickly. This proactive approach leads to more robust and reliable
  software.

- **Refactor more frequently:** The constant integration and deployment cycle
  makes it easier to identify and address technical debt. Small, frequent
  refactoring becomes a natural part of the development process, leading to
  cleaner and more maintainable code.

In essence, trunk-based development fosters a culture of quality by making
developers feel more connected to the production environment. This increased
sense of ownership and responsibility translates into higher-quality code, fewer
bugs, and ultimately, a better product.

## The Golden State

The ultimate goal, the "Golden State" we envision with Trunk-Based Development
and robust automation within the Salesforce ecosystem, is a system where
engineers merge their code to the `main` branch only once. From that point
forward, the system automatically takes over, handling the promotion of those
changes to higher environments on a pre-determined cadence, such as weekly.

What does this look like?

<!-- prettier-ignore-start -->
<div class="plantuml-container">
<!-- prettier-ignore-end -->

<!-- prettier-ignore-start -->
{% plantuml %}
@startuml

scale 1.5
skinparam sequenceArrowThickness 2.5


skinparam cloud<<cloudStyle>> {
  BackgroundColor #3baed3
  FontColor #ffffff
}

skinparam card<<branch>> {
  BackgroundColor #FF0000
  FontColor #ffffff
}


hide stereotype

folder "Developer's Responsibility" {
  cloud Scratch <<cloudStyle>>
  card Feature_branch as "<&fork> **feature/xyz**" <<branch>>
}

folder "System's Responsibility" {
  cloud QA <<cloudStyle>>
  cloud UAT <<cloudStyle>>
  cloud Production <<cloudStyle>>

  card main_branch as "<&fork> **main**" <<branch>>
  card release_next_branch as "<&fork> **release_next** \n (twice per week)" <<branch>>
  card release_active_branch as "<&fork> **release_active** \n (once per week)" <<branch>>
}

Scratch -[#black,dashed]u-> Feature_branch : " commit"
Feature_branch -[#black]r-> main_branch : ""
main_branch -[#black]r-> release_next_branch: ""
release_next_branch -[#black]r-> release_active_branch : ""

main_branch -[#orange]d-> QA
main_branch -[#green]d-> QA
main_branch -[#orange,dashed]-> UAT

release_next_branch -[#orange]d-> UAT
release_next_branch -[#green]d-> UAT
release_next_branch -[#orange,dashed]-> Production

release_active_branch -[#orange]d-> Production
release_active_branch -[#green]d-> Production


legend
  | Arrow | Description |
  | <color:black><size:18><&arrow-right></size></color>   | Pull request / Branch Creation |
  | <color:orange><size:18><&arrow-right></size></color>  | Validation |
  | <color:green><size:18><&arrow-right></size></color>   | Deployment |
  | <color:orange><size:18>- <&arrow-right></size></color>   | Validation on merge |
  
endlegend

@enduml
{% endplantuml %}
<!-- prettier-ignore-end -->

<!-- prettier-ignore-start -->
</div>
<!-- prettier-ignore-end -->

The "Golden State" depicted above resembles our previous diagram but shifts the
emphasis from manual actions by developers to automated processes managed by the
system.

Here's the key difference: after a developer merges their feature branch into
`main`, the system takes over. Features are automatically promoted through QA,
UAT, and finally production on a pre-defined schedule. This automated promotion
occurs by creating a new `release_next` branch from `main` twice per week. Then,
once per week, a new `realease_active` branch is created from `release_next`.

This **"merge once, then walk away"** paradigm allows developers to focus on
what they do best: building features. They no longer need to worry about the
complexities of manual deployments, regular cherry-picking, or reconciling
environment drift. The automated promotion significantly reduces the cognitive
load on developers, freeing them to move on to their next task without the
overhead of managing deployments. It also drastically reduces the risk of human
error associated with manual processes.

#### Eager Validation

In this workflow, whenever changes are merged into `main`, the contents of main
are validated in the `UAT` sandbox. Similarly the contents of the `release_next`
branch are validated in production upon merging. This eager validation is a key
component of the trunk based development model for Salesforce; it ensures that
as changes are promoted, they are valid within the next target environment. If
this validation ever fails, a ticket should be created and assigned to the
oncall engineer to investigate. This saves many headaches by proactively
identifying issues with compilation or test execution multiple days before the
deployment is scheduled to occur.

#### Scheduled Branch promotion

One powerful way to achieve this automation is through GitHub Actions, a
platform for automating your development workflow directly within your
repository.

Here's an example YAML configuration that demonstrates how to schedule the
promotion of changes from `main` to `release_next`:

```yml
name: Update Release Next Branch

on:
  schedule:
    - cron: "0 14 * * 0,4" # Runs at 2 PM (14:00) UTC every Sunday (0) and Thursday (4)

jobs:
  update_release_next:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Validate Contents in Salesforce
       run: |
        # perform validation in target org before merging

      - name: Update Release Next
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"

          git checkout release_next
          git reset --hard main # Or use git merge main if you prefer merging

          git push -f origin release_next  # Use -f with caution! Consider git push origin release_next for merges.
```

#### Hotfix Handling

Even in this automated system, there's room for flexibility. In rare cases where
urgent fixes are needed, teams can still cherry-pick specific code changes into
appropriate branches/environments (e.g., a hotfix for a critical bug). This
"hotfix" mechanism acts as a safety net, allowing for rapid responses to
unexpected issues.

## Why is Trunk-Based Development Challenging in Salesforce?

While Trunk-Based Development offers significant advantages, implementing it
within the Salesforce ecosystem presents unique challenges, both cultural and
technical.

### Cultural Challenges

A significant cultural hurdle stems from established practices. Many Salesforce
teams are accustomed to a one-to-one mapping between a branch and an org
(sandbox or production). This implies that each branch acts as a distinct and
independent copy of the codebase. This practice creates multiple divergent
sources of truth, which is fundamentally at odds with the core principles of how
engineers in other software development teams utilize source control. Shifting
away from this ingrained mindset requires a significant change in team workflows
and understanding.

### Technical Challenges

Beyond the cultural aspects, there are also key technical requirements that must
be addressed to successfully implement Trunk-Based Development in Salesforce.
The most crucial of these is ensuring that the contents of the `main` branch are
_always safely deployable_ to multiple Salesforce environments. This
necessitates several key capabilities:

#### Robust Environment Variable Management

Because `main` is deployed to different environments (e.g., QA, UAT,
production), the codebase must be able to adapt to environment-specific
configurations. This means having a robust mechanism for managing environment
variables that control things like API endpoints, email addresses, and other
environment-specific settings. These variables must be injected into the
deployment process without requiring changes to the codebase itself.

#### Effective Feature Flagging

In Trunk-Based Development, features are often merged into `main` before they
are fully ready for release. To prevent these “half-baked” features from
negatively impacting the user experience, teams need a way to selectively enable
or disable them. Changes which are not ready for prime-time are to be safely
deployed to the target application environment, but hidden from visibility or
throw a nicely rendered error until the feature is ready.

Overcoming these cultural and technical challenges is essential for unlocking
the full potential of Trunk-Based Development in Salesforce. The following
sections will explore strategies and best practices for addressing these
challenges and successfully implementing this powerful development model.

## Environment Variables

Environment variables are crucial to the success of a trunk-based development
workflow, as they allow the contents of a single branch (`main`) to be safely
deployed to multiple target environments. Without a robust mechanism for
managing environment-specific configurations, deploying the same codebase to
development, QA, UAT, and production would be impossible.

In the Salesforce context, "environment variables" often translate to XML
modifications performed on the metadata before it is deployed to a target
environment. Many metadata types require environment-specific settings,
including workflow outbound message endpoint URLs, email alerts, connected app
consumer secrets, and many more. Generally speaking, we need a solution that
isn't bound to the schema of a specific metadata type but rather supports
generic XML modifications.

Salesforce's existing metadata string replacements are often insufficient to
meet our needs. Some transformations are non-trivial string replacements,
requiring multiple nodes within the XML structure to be modified. When using
primitive string find and replace, issues can be encountered, particularly when
dealing with whitespace and newlines within the XML. These subtle differences
can lead to unexpected deployment failures or, worse, subtle bugs in production.

The ideal solution involves a set of environment configuration files within the
project, living within a `./environments` folder. The specific implementation
details of the configuration file format are not critical – they could be JSON,
YAML, Textproto, or any other suitable format. However, each instance of an
environment replacement should contain four key pieces of information:

- **Applicable environment identifier:** A label that identifies the target
  environment (e.g., `QA`, `UAT`, `Production`).
- **File path of interest:** The relative path to the metadata file within the
  Salesforce project that needs modification.
- **XPath to node of interest:** An XPath expression that precisely identifies
  the XML node(s) to be modified.
- **String replacement for the entire node:** The complete XML snippet that
  should replace the identified node(s).

When environment variables are set, the nodes within the designated file,
identified by the XPath, are fully replaced by their corresponding value in the
configuration file. This simple yet powerful mechanism enables support for all
of Salesforce's XML-based metadata types and is easy to read and maintain.

Here's an example of such a configuration file (using YAML):

```yaml
environment_values:
  QA:
    - file: workflows/MyWorkflow.workflow-meta.xml
      xpath: //WorkflowOutboundMessage[fullName='Demo']/endpointUrl
      replacement: <endpointUrl>https://qa.example.com/endpoint</endpointUrl>
    - file: email/MyEmailAlert.email-meta.xml
      xpath: //WorkflowAlert[fullName='Demo']/senderAddress
      replacement: <senderAddress>qa-alerts@example.com</senderAddress>
  UAT:
    - file: workflows/MyWorkflow.workflow-meta.xml
      xpath: //WorkflowOutboundMessage[fullName='Demo']/endpointUrl
      replacement: <endpointUrl>https://uat.example.com/endpoint</endpointUrl>
    - file: email/MyEmailAlert.email-meta.xml
      xpath: //WorkflowAlert[fullName='Demo']/senderAddress
      replacement: <senderAddress>uat-alerts@example.com</senderAddress>
  Production:
    - file: workflows/MyWorkflow.workflow-meta.xml
      xpath: //WorkflowOutboundMessage[fullName='Demo']/endpointUrl
      replacement: <endpointUrl>https://production.example.com/endpoint</endpointUrl>
    - file: email/MyEmailAlert.email-meta.xml
      xpath: //WorkflowAlert[fullName='Demo']/senderAddress
      replacement: <senderAddress>production-alerts@example.com</senderAddress>
```

This configuration clearly defines the environment-specific values for the
specified metadata components, ensuring that each deployment uses the correct
settings. This approach eliminates the need for manual modifications and
significantly reduces the risk of errors associated with traditional string
replacement methods.

## Feature Flagging

When working in a trunk-based development workflow, any metadata checked into
the repository _will_ eventually go to production. This means that many features
will be "half-baked" for a while, deployed but inactive, until they are ready
for use. This is where feature flagging comes into play.

Our objective is to decouple the business go-live from the technical go-live.
Components should be deployed to the target Salesforce environment and sit
dormant, then activated at a later date, independent of the deployment schedule.
This allows for continuous integration and continuous delivery, even when
features are not yet ready for prime time.

The primary strategy for achieving this is to gate the injection point for the
feature you are working on behind a custom permission. The custom permission
acts as a feature flag. A recommended naming convention is to prefix custom
permissions used as feature flags with `FF_`, for example,
`FF_NewCustomerPortal` or `FF_EnhancedSearch`. This clear naming convention
makes it easy to identify feature flags within your Salesforce org.

Your application logic must be written in such a way that behavior is bifurcated
based on whether the user has the custom permission assigned. This can be done
in Apex, Visualforce, Lightning Web Components, or any other part of your
Salesforce application.

Here are some examples of how you might implement feature flagging in different
contexts:

**Apex:**

```java
if (FeatureManagement.isFeatureEnabled('FF_NewCustomerPortal')) {
    // Code to execute when the feature is enabled
    PageReference newPage = new PageReference('/newCustomerPortal');
    newPage.getParameters().put('id', controller.getId());
    newPage.setRedirect(true);
    return pageRef;
} else {
    // Code to execute when the feature is disabled
    return Page.OldCustomerPortal;
}
```

**Lightning Web Component (LWC) JavaScript:**

```javascript
import { LightningElement, api } from "lwc";
import hasPermission from "@salesforce/userPermission/FF_EnhancedSearch";

export default class MyComponent extends LightningElement {
  @api recordId;

  get showNewFeature() {
    return hasPermission;
  }
}
```

**Lightning Web Component (LWC) HTML:**

```html
<template>
  <template if:true="{showNewFeature}">
    <c-enhanced-search record-id="{recordId}"></c-enhanced-search>
  </template>
  <template if:false="{showNewFeature}">
    <c-basic-search record-id="{recordId}"></c-basic-search>
  </template>
</template>
```

We prefer using custom permissions for feature flagging because they offer broad
accessibility throughout the entire Salesforce platform. This versatility allows
us to control feature visibility and behavior in a multitude of contexts,
ensuring consistent and reliable feature gating. Custom permissions can be
checked in a wide range of places, including:

- **Apex:** As shown in the previous example, Apex code can easily check for the
  presence of a custom permission using the
  `FeatureManagement.isFeatureEnabled()` method or by querying the
  `PermissionSetAssignment` and `PermissionSet` objects directly. This allows
  for dynamic control of Apex logic based on the feature flag.

- **Lightning Web Components (LWC):** Both JavaScript and HTML within LWC can
  utilize custom permissions. JavaScript can use the
  `@salesforce/userPermission` wire adapter or similar methods to determine if a
  user has a specific permission. HTML templates can use the `if:true` and
  `if:false` directives in conjunction with the permission check to
  conditionally render elements.

- **Page Builder Visibility Criteria:** Lightning App Builder allows you to set
  visibility criteria for components based on permissions. This means you can
  show or hide entire sections of a page based on feature flags, providing a
  seamless user experience.

- **Flows:** Flows can use Decision elements to branch logic based on whether a
  user has a specific custom permission. This makes it possible to create
  dynamic flows that adapt to different feature sets.

- **Validation Rules:** You could even use custom permissions in validation
  rules to enforce different data entry rules based on feature access.

This pervasive availability of custom permissions across the Salesforce platform
makes them an ideal choice for feature flagging. They provide a centralized and
consistent way to manage feature access.

When this is properly implemented, the activation event becomes the assignment
of a permission set containing the custom permission. You would create a
permission set, for example, `NewCustomerPortal_Access`, and add the
`FF_NewCustomerPortal` custom permission to it. When the business is ready to
launch the feature, simply assign the `NewCustomerPortal_Access` permission set
to the appropriate users. This enables easy deployments and, just as
importantly, easy rollbacks should something go wrong. Deactivating a feature is
as simple as removing the permission set assignment. This provides a clean and
controlled way to manage feature releases and minimizes the impact of any
unforeseen issues.

## Conclusion

Trunk-based development offers a compelling alternative to traditional branching
strategies in Salesforce development. By embracing frequent integration,
automated deployments, and feature flagging, teams can significantly improve
their development velocity, reduce risk, and enhance software quality. The
"merge once, then walk away" paradigm empowers developers to focus on building
great features, while the system handles the complexities of deployment and
promotion. While implementing trunk-based development within the Salesforce
ecosystem presents unique challenges, the benefits far outweigh the initial
effort required to overcome them.

The shift to trunk-based development requires a cultural change, a commitment to
automation, and careful planning. It's not an overnight transformation. However,
the journey towards a more streamlined and efficient development process begins
with a single step.

I challenge you to consider how trunk-based development could benefit your
Salesforce team. Start small. Perhaps pilot the approach with a less critical
project or a single team. Experiment with the techniques described in this post.
Explore different feature flagging strategies, and refine your environment
variable management. Don't be afraid to iterate and adjust your approach as you
learn what works best for your organization.

The potential rewards are substantial: faster release cycles, improved code
quality, and happier developers. Embrace the trunk, and unlock the full
potential of your Salesforce development team.

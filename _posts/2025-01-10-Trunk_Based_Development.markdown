---
layout: post
mathjax: true
title: "There Can Be Only One (Branch): Trunk-Based Development for Salesforce"
date: 2025-01-10 01:43:24 -0500
---

## Trunk Based Development

In the Salesforce development world, a common practice has been to rely on
persistent, long-lived branches for non-production environments like
development, testing, and staging. While seemingly straightforward, this
branching strategy often introduces a host of challenges that can significantly
hinder development velocity and release cadence. From the ever-present threat of
merge conflicts and the insidious creep of environment drift, to the
complexities of deployment and the frustrating repetition of steps within the
development pipeline, these persistent branches can become a major bottleneck
for your Salesforce development team. This post explores the pitfalls of this
traditional branching approach within Salesforce and introduces a more
streamlined and efficient alternative: Trunk-Based Development.

### The Status Quo

The prevailing branching strategy in many Salesforce development teams often
resembles the following pattern: Developers work on individual features within
dedicated feature branches, a practice that, in isolation, has merit. However,
the issues arise when these branches interact with the broader environment
structure. Typically, teams maintain three persistent branches: `QA`, `UAT`, and
`main`. Each of these branches serves as an independent source of truth,
directly tied to corresponding sandboxes and the production environment. This
means the `QA` branch is the source for the QA sandbox, `UAT` for the UAT
sandbox, and `main` for production.

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
branching off for each release, a specific commit on `main` is chosen as the
release point at a scheduled cadence. This means that a historical point in the
commit history of `main` becomes the basis for the release. This approach
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
f5 -[LIGHT_BLUE]r-> c7

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

## The Golden State

The ultimate goal, the "Golden State" we envision with Trunk-Based Development
and robust automation within tghe Salesforce ecosystem, is a system where
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

release_next_branch -[#orange]d-> UAT
release_next_branch -[#green]d-> UAT

release_active_branch -[#orange]d-> Production
release_active_branch -[#green]d-> Production


legend
  | Arrow | Description |
  | <color:black><size:18><&arrow-right></size></color>   | Pull request / Branch Creation |
  | <color:orange><size:18><&arrow-right></size></color>  | Validation |
  | <color:green><size:18><&arrow-right></size></color>   | Deployment |
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

#### Hotfix Handling: A Safety Net

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

Because `main` is deployed to different environments (e.g., development, QA,
UAT, production), the codebase must be able to adapt to environment-specific
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

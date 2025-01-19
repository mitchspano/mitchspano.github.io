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
structure. Typically, teams maintain three persistent branches: "QA," "UAT," and
"main." Each of these branches serves as an independent source of truth,
directly tied to corresponding sandboxes and the production environment. This
means the "QA" branch is the source for the QA sandbox, "UAT" for the UAT
sandbox, and "main" for production.

The process of promoting changes up the environments involves merging feature
branches into "QA." Critically, when a pull request (PR) containing these
changes is created or updated, the scope of the changes is validated against the
target Salesforce environment (in this case, the QA sandbox). This validation
might include running Apex tests, checking for code coverage, and performing
static code analysis. Once the PR is approved and merged into the "QA" branch,
the changes are then deployed to the QA sandbox. From there, changes are often
cherry-picked (or sometimes merged) across the branches – from "QA" to "UAT,"
and finally from "UAT" to "main." Each promotion up the chain initiates its own
validation and deployment in the corresponding environment and the processes is
repeated n times for n environments.

The "QA", "UAT", and "main" branches are not ephemeral; they are
ever-persistent, extending throughout the project's entire lifespan. This
longevity creates several problems and the multi-step process, while seemingly
controlled, introduces significant overhead and risk, as we'll explore further
in this post.

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

cloud Scratch <<cloudStyle>>
cloud QA <<cloudStyle>>
cloud UAT <<cloudStyle>>
cloud Production <<cloudStyle>>

card Feature_branch as "<&fork> **feature/xyz**" <<branch>>
card QA_branch as "<&fork> **QA**" <<branch>>
card UAT_branch as "<&fork> **UAT**" <<branch>>
card main_branch as "<&fork> **main**" <<branch>>

Scratch -[#black,dashed]u-> Feature_branch : " commit"
Feature_branch -[#black]r-> QA_branch : "merge"
QA_branch -[#black]r-> UAT_branch : "🍒⛏️"
UAT_branch -[#black]r-> main_branch : "🍒⛏️"

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

### Issues with the Status Quo

While the described branching strategy might appear organized at first glance,
it introduces several significant issues that can severely hamper a Salesforce
development team's efficiency and reliability. One of the most glaring problems
is the constant repetition of steps. Promoting changes from development to QA,
then to UAT, and finally to production requires developers to perform the same
actions multiple times. They must manually invoke the cherry-pick process for
every change moving between branches. This manual intervention is not only
time-consuming but also prone to human error.

This manual cherry-picking process also leaves the project ripe for creating
drift between environments. Because changes are manually selected and applied to
each branch, it's easy to accidentally omit a crucial component or introduce
inconsistencies. Over time, these small discrepancies accumulate, leading to
significant differences between the QA, UAT, and production environments.
Reconciling this drift becomes a major undertaking, often requiring extensive
debugging and rework, particularly when trying to diagnose production issues
that can't be easily reproduced in lower environments.

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

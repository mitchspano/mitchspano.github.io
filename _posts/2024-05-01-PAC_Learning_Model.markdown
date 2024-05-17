---
layout: post
mathjax: true
tikzjax: true
title: "Decoding PAC: The Backbone of Reliable Predictions"
date: 2024-05-01 01:43:24 -0500
---

## A Formal Learning Model

In some of the recent posts on this blog, we have discussed the concept of a _learner_ which
consumes a labeled data set and produces a hypothesis function which can be used to predict the
labels for previously unseen data.

In ths post, we will formalize this model and introduce some foundational concepts which will
enable us to prove statements about the progressive improvement of a learner as it
observes more and more data.

### Learner's Responsibility

Recall that our training data set is defined as:

$$
  S = \left( \left(x_1, y_1\right)...\left(x_n, y_n\right)\right) \in \mathcal{X} \times \mathcal{Y}
$$

The learner is requested to generate a _hypothesis_ $h: \mathcal{X} \to \mathcal{Y}$ that can be used to predict the label for a piece of previously unseen data.

There are many such potential hypotheses. The set of all potential hypotheses is called
(unsurprisingly) a _hypothesis class_ $\mathcal{H}$.

Let's consider the order of operations for the learner to produce a hypothesis. The learner
observes a data point and produces a hypothesis that is consistent with the data it's seen
thus far. Then, the labels provided by the "teacher" inform the learner if it was correct,
or if it made a mistake. If the learner makes a mistake, the learner updates its internal state
to account for the newly gained knowledge.

<!-- prettier-ignore-start -->
{% plantuml %}
@startuml
participant "Learner" as L
participant "Teacher" as T

hnote over L : Observe Data Point
L -> T: Produce guess for label
T ->o L: Correct

hnote over L : Observe Data Point
L -> T: Produce guess for label
T -> L: Mistake
destroy L
L -> L: Update Internal State

@enduml
{% endplantuml %}
<!-- prettier-ignore-end -->

Every time the learner updates its state, it is generating a new hypothesis.
The general idea is that if we can construct an algorithm which follows this procedure, we should
be able to produce hypotheses of higher and higher quality. This is the foundation of machine
learning; using the data we have already observed to improve the predictive power of our produced
hypothesis function.

### Probably Approximately Correct Learning Model

When we talk about predicting a label for a piece of data when given a training data set, we
make the assumption that there exists some underlying function $f: \mathcal{X} \to \mathcal{Y}$
which perfectly labels our training data and all possible elements in the domain set.

It is extremely unlikely that we will stumble upon this exact function perfectly with the data
that we observe in the real world, so we relax the definition of success to something that is
"good enough" in layman's terms. More formally, we want to produce a hypothesis which is
**Probably Approximately Correct**.

There are two parts of this definition:

#### Approximately Correct ($\epsilon$)

This part of the definition means that the model, or hypothesis, that the algorithm learns doesn't
need to be perfect. Instead, it should be close enough to the true model. This is measured by an
error rate, which should be small. In mathematical terms, this error rate is represented by
$\epsilon$ (epsilon). Epsilon is a small positive number $ \epsilon \in [0,1]$ that sets the maximum allowable error rate for the hypothesis. A smaller epsilon means the hypothesis needs to be more accurate

#### Probably ($\delta$)

This part implies that while the hypothesis needs to be approximately correct, it does not need to be so under all possible scenarios, but rather with a high probability. In other words, there's still a chance, albeit small, that the hypothesis may not meet the error threshold set by epsilon.

The delta ($\delta$) parameter specifies how confident we want to be in our hypothesis. It is the probability that the hypothesis might not meet the accuracy specified by epsilon. A smaller delta means higher confidence in the hypothesis being within the error threshold.

### Mathematical Definitions

#### Training Error

Our hypothesis function $h$ is likely not going to be perfect every time. We define the
_training error_ as the error the classifier incurs over the training sample:

$$
L_S(h) \stackrel{\text{def}}{=} \frac{|\{ i \in [m] : h(x_i) \neq y_i \}|}{m}
$$

This reads as:

> The loss over the training set $S$ of the hypothesis function $f$ for $m$ data points
> is equal by definition to the quantity of all data points that were not properly classified
> by the hypothesis divided by $m$.

It is simply the ratio of incorrectly classified points by the hypothesis function $h$.

#### Realizability Assumption

Let $\mathcal{D}$ be the unknown distribution that our sample set $S$ is drawn from,
and $f$ the unknown labeling function we are trying to learn.

The _realizability assumptions_ states that there exists $h^* \in \mathcal{H}$ such that
$L_{(\mathcal{D}, f)}(h^*) = 0$.

Simply put, a perfect hypothesis function exists.

#### Probably Approximately Correct (PAC)

The overall objective is to produce a function which decides how many examples
or pieces of data $m$ we need to look at to be confident in the predictions made by our chosen
method or formula. It uses two parameters, $\epsilon, \delta$ which help set the level of
confidence and accuracy.

If we can produce a algorithm that hones in on an method from the hypothesis class $\mathcal{H}$
that we are confident ($\delta$) is quite accurate ($\epsilon$) after seeing enough examples,
then the hypothesis class $\mathcal{H}$ is PAC learnable.

Here is the formal (and very mathematically dense) definition of learnability:

A hypothesis class $\mathcal{H}$ is PAC learnable if there exist a function
$m_{\mathcal{H}} : (0,1)^2 \rightarrow \mathbb{N}$ and a learning algorithm with
the following property: For every $\epsilon, \delta \in (0,1)$, for every distribution
$\mathcal{D}$ over $\mathcal{X}$, and for every labeling function
$f : \mathcal{X} \rightarrow \{0,1\}$, if the realizable assumption holds with
respect to $\mathcal{H}$, $\mathcal{D}$, $f$, then when running the learning
algorithm on $m \geq m_{\mathcal{H}}(\epsilon, \delta)$ of independently
and identically distributed examples generated
by $\mathcal{D}$ and labeled by $f$, the algorithm returns a hypothesis $h$ such
that, with probability of at least $1 - \delta$ (over the draws from $\mathcal{D}$)
, $L_{(\mathcal{D}, f)}(h) \leq \epsilon$.

## Learning a Field Goal

Imagine you are a kicker on a football team. You are practicing.

<script type="text/tikz">
\begin{tikzpicture}
  \draw[latex-latex] (-5,0) -- (5,0);
  \draw[dashed] (-2, -1) -- (-2, 1);
  \draw[dashed] (-1, -1) -- (-1, 1);
  \draw[dashed] (1, -1) -- (1, 1);
  \draw[dashed] (2, -1) -- (2, 1);

  \fill[red] (-3.743, 0) circle (3pt);
  \fill[red] (-2.977, 0) circle (3pt);
  \fill[red] (-2.915, 0) circle (3pt);
  \fill[red] (-2.548, 0) circle (3pt);
  \fill[red] (-2.062, 0) circle (3pt);

  \fill[blue] (-0.906, 0) circle (3pt);
  \fill[blue] (-0.537, 0) circle (3pt);
  \fill[blue] (0.428, 0) circle (3pt);
  \fill[blue] (0.823, 0) circle (3pt);
  \fill[blue] (0.906, 0) circle (3pt);

  \fill[red] (2.062, 0) circle (3pt);
  \fill[red] (2.269, 0) circle (3pt);
  \fill[red] (2.796, 0) circle (3pt);
  \fill[red] (3.496, 0) circle (3pt);
  \fill[red] (3.718, 0) circle (3pt);

\end{tikzpicture}
</script>

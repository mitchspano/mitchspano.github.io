---
layout: post
mathjax: true
title: "Beyond Buzzwords: Back to the Basics of Machine Learning"
date: 2024-03-19 19:43:24 -0500
---

## It Seems Like Everyone's Talking About Artificial Intelligence...

In today's fast-paced technological landscape, it seems like everyone is talking about artificial intelligence (AI). From news headlines to everyday conversations, the buzz surrounding AI is impossible to ignore. While much of this discourse centers around the flashy applications and breakthroughs in the field, there's an underlying force that drives the remarkable capabilities of AI: machine learning.

## Very Few Actually Know What They're Talking About.

Amidst the fascination with large language models and neural networks, the fundamental mathematical principles underpinning machine learning often get overshadowed. In this blog post, we're diving into the mathematical foundations of machine learning, exploring the definitions that form the backbone of this technology.

So, let's venture beyond the buzzwords and unveil the true underpinnings of machine learning – it's time to get back to the basics behind the AI phenomenon.

#### Credit

Most of the examples that I will be sharing in this post and beyond are heavily inspired from the book ["Understanding Machine Learning: From Theory to Algorithms"](https://www.cs.huji.ac.il/~shais/UnderstandingMachineLearning/understanding-machine-learning-theory-algorithms.pdf) by Shai Shalev-Shwartz and Shai Ben-David.

## Tasty Apples 🍎

Let's just say that we own an apple orchard.
We would like to determine which apples are likely to be tasty and which ones are not tasty.
Now, obviously we cannot take a bite out of each apple before we sell them to a customer, but
there are certain things we can measure which might impact the tastiness (or lack thereof) of an
apple. There are plenty of features we could measure: species, color, firmness, density, weight, age, etc. but for sake of example, let's just say that we are limiting ourselves to
only consider the color and firmness of the apple.

Now let's imagine that as customers are leaving the orchard with their newly purchased apples,
we write down the color as a decimal value from 0 to 1 (0 being dead and brown and 1 being bright red) and the firmness a decimal value from 0 to 1 (0 being soft and squishy and 1 being hard as a rock). We then ask the customers to take a bite out of the apple and tell us if it is tasty or not and we record their response.

What we would like to do is use this data that we have gathered from surveying our customers to predict which apples that we sell in the future will be tasty or not, without us having to ask the customers or take a bite ourselves.

We can think of the input here being the features of a single apple and the output that we would like to produce is a binary label: tasty or not tasty.

In this post, we will explore fundamental concepts in machine learning referring to the to Apple tastiness use case. This example will enable us to define formally some of the key concepts of Machine Learning.

#### ⚠️ Warning - Mathematics Coming Up ⚠️

**Some people will say that you don't need to know mathematics to understand Artificial Intelligence; those people are lying to you**. You need to understand some core mathematical constructs including Set theory, Probability, Calculus, and Linear Algebra. I will do my best to cover these topics as gently as I can, but I _strongly_ recommend brushing up on these if you want to be successful in your machine learning journey.

## Vocabulary

### General Mathematics Vocabulary

#### Set

A set, in mathematics, is a collection of distinct objects, considered as a single entity. Sets are often denoted as a single capital letter such as $\mathcal{S}$. These objects can be anything: numbers, letters, symbols, or even other sets. Each object within a set is called an element or a member. Sets are defined solely by their elements; the order in which elements are listed does not affect the set itself.

Set notation is a standardized method used to represent sets, providing a concise and formalized way to describe their elements and properties. The most common notation for denoting a set is by enclosing its elements within curly braces $\\{ \\}$. For example, the set of natural numbers less than 5 can be denoted as $\\{1, 2, 3, 4\\}$.

Set builder notation defines a set by specifying a rule or condition that its elements must satisfy. It typically takes the form $\\{x \vert \text{ condition}\\}$, where $x$ represents the elements and "condition" represents the rule. For instance, the set of all even numbers can be represented as $\\{x \vert x \text{ is even}\\}$.

#### Membership

When discussing sets, we often talk about something as being a member of the set or not. For example, set of natural numbers less than 5 can be denoted as $ \mathcal{S} = \\{1, 2, 3, 4\\}$. To make a statement about an item $x$ being an element of the set, we use the notation $x \in \mathcal{S}$ which reads as "the element $x$ is **in** the set $\mathcal{S}$". Similarly, we use the notation $x \notin \mathcal{S}$ which reads as "the element $x$ is **not in** the set $\mathcal{S}$".

#### Cartesian Product

The cross of two sets, also known as the Cartesian product, is a fundamental operation in set theory that produces a new set consisting of all possible ordered pairs formed by taking one element from each of the original sets. Given two sets A and B, their Cartesian product, denoted as $A \times B$, is defined as:

\begin{equation}
\nonumber
\begin{aligned}
A \times B = \\{ (a,b) | a \in A, b\in B \\}
\end{aligned}
\end{equation}

In other words, the Cartesian product $A \times B$ contains all possible pairs $(a, b)$ where $a$ is an element of set $A$ and $b$ is an element of set $B$. The order of the elements in the pairs is significant; so $(a, b)$ is distinct from $(b, a)$ if $a \ne b$ or if $A \ne B$.

#### Functions

A function is a mathematical relation between two sets, known as the domain and the range, that assigns to each element in the domain exactly one element in the range. In simpler terms, a function is a rule or mapping that associates each input value (or argument) from the domain with exactly one output value in the range.

More formally, let $\mathcal{X}$ and $\mathcal{Y}$ be two sets. A function $f$ from $\mathcal{X}$ to $\mathcal{Y}$, denoted as $f: \mathcal{X} \to \mathcal{Y}$, is a correspondence that assigns to each element
$x$ in the domain set $\mathcal{X}$ a unique element $y$ in the range set $\mathcal{Y}$. Mathematically, this correspondence is represented by the functional notation:

$$ f(x) = y$$

### Machine Learning Vocabulary

#### Domain set

An arbitrary set, $\mathcal{X}$ . This is the set of objects that we may wish to label. For example, in the apple tastiness problem mentioned before, the domain set will be the set of all apples.

Usually, these domain points will be represented by a vector of features such as species, color, firmness, density, weight, age, etc.

#### Label Set

For our current discussion, we will restrict the label set to be a two-element set, usually $ \\{ 0, 1 \\} $. Let $\mathcal{Y}$ denote our set of possible labels. For our apple example, let $\mathcal{Y}$ be $\\{0, 1\\}$, where 1 represents being tasty and 0 means not-tasty.

#### Training Data

A finite sequence that is a subset of the domain set paired with corresponding labels.

$$S = \left( \left(x_1, y_1\right)...\left(x_n, y_n\right)\right) \in \mathcal{X} \times \mathcal{Y}$$

This serves as the basis for instructing our program to recognize patterns indicative of tasty apples.

#### Learner

Finally, the learner, often an algorithm or model, undergoes training on the labeled dataset to discern underlying patterns and subsequently make predictions on new, unseen data. Formally we say that the learner is requested to generate a _prediction rule_ or _hypothesis_ $h: \mathcal{X} \to \mathcal{Y}$. The predictor can be used to predict the label of new domain points. In our apples example, the learner will predict whether future apples he examines in the orchard are going to be tasty or not. We use the notation $A(S)$ to denote the hypothesis that a learning algorithm, $A$, returns upon receiving the training sequence $S$.

#### Accuracy

It's great that we can produce a hypothesis, but how do we measure its accuracy to evaluate it against other hypotheses? Assume $f:\mathcal{X} \to \mathcal{Y}$ exists and is an unknown function we are trying to predict. We measure the _error rate_ of our hypotheses as the probability that it did not predict the correct label for a random element in our domain. Formally speaking, the error rate $L$ of a hypothesis $h$ is defined as:

$$L(h) = \mathbb{P}[h(x) \ne f(x)]$$

### Apple-ication

Imagine we had gathered the following data from our customers about the apples they had eaten:

|     | Color | Firmness | Tasty? |
| --- | ----- | -------- | ------ |
| 1   | 0.9   | 0.7      | 1      |
| 2   | 0.4   | 0.3      | 0      |
| 3   | 0.9   | 0.2      | 0      |
| 4   | 0.6   | 0.6      | 1      |
| 5   | 0.5   | 0.3      | 0      |

Let's formalize this scenario with the vocabulary learned above.

The _Domain Set_ would be the set of all apples.

The _Label Set_ is the values in the "Tasty?" column.

Each element in our training data consists of a pair of color and firmness ratings, combined with the tasty label:

$$S = \left( \left(\left(0.9,0.7\right), 1\right)...\left(\left(0.5,0.3\right), 0\right)\right)$$

Let's assume our learner produced the following hypothesis - an apple is tasty if the sum of its color and firmness is greater than or equal to one.

$$
h(c,f) = \begin{cases}
1 & : c + f \geq 1 \\
0 & : \text{otherwise}
\end{cases}
$$

We would see that this hypothesis is correct for data points 1,2,4 and 5, but incorrect for data point number 3. This means that the _Error Rate_ for our hypothesis on this data set would be $\frac{1}{5} = 20\%$

## What Comes Next?

Now that we've established a solid understanding of foundational concepts in machine learning, such as set notation and the definition of a function, we are equipped to delve deeper into the mechanisms of learning itself. By formalizing these ideas, we lay the groundwork for developing concepts and tools that enable machines to generate valuable predictions. Stay tuned for future posts in the blog as we explore deeper and deeper down the rabbit hole of Machine Learning.

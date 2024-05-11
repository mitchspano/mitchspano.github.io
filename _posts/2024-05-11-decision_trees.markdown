---
layout: post
mathjax: true
title: "Shallow Learning Roots: Decision Trees"
date: 2024-05-11 01:43:24 -0500
---

## A Beginner's Guide to Decision Trees

In the [last post](back-to-basics-ml), we discussed the concept of a _learner_ which undergoes training on the labeled
data set $S$ to discern underlying patterns and subsequently make predictions on new, unseen data.

In ths post, we will go over a classic example of a shallow learning algorithm for binary classification - **Decision Trees**.

A decision tree is a hypothesis function, $h:\mathcal{X} \to \mathcal{Y}$, that predicts the
label associated with an instance $x$ by traveling from a root node of a tree to a leaf, evaluating
decision criteria along the way.

At each node of the path from the root node to the leaf, the next child is chosen on a basis of
partitioning or splitting the training data.

Today we focus on the binary classification setting where $\mathcal{Y} = \\{0, 1\\}$ but decision
trees can be applied for other prediction problems as well.

Here is an example of a simple decision tree. At each node, going to its left subtree denotes making a "no" (or 0) decision,
going to the right subtree denotes a "yes" (or 1) decision. Terminal states are denoted in red and green colors:

<!-- prettier-ignore-start -->
{% plantuml %}
@startwbs
<style>
wbsDiagram {
  .green {
      BackgroundColor #90EE90
  }
  .red {
      BackgroundColor #FF474C
  }
}
</style>
+ Am I hungry?
++ STOP <<red>>
++ Am I under my calorie limit for the day?
++- Have I worked out today?
+++- STOP <<red>>
++++ EAT <<green>>
+++ EAT <<green>>

@endwbs
{% endplantuml %}
<!-- prettier-ignore-end -->

## Underwiting Example

Imagine you are a banker, and your underwriting team processes
many thousands of applications for loans throughout the year.

The team spends a lot of time qualifying these applicants and you would like to use what you have
learned from prior applicants to help qualify new potential applicants.

You have the following attributes available from prior applicants:

- **Credit Score Good (1 = good, 0 = bad)**: A typical threshold might be a credit score above 650 is considered good.
- **Employment Status (1 = employed, 0 = unemployed)**: Employment status can indicate financial stability.
- **Existing Debts (1 = no existing debts, 0 = has existing debts)**: The absence of existing debts could
  indicate better financial health.
- **Home Ownership (1 = owns home, 0 = does not own home)**: Owning a home might reflect financial
  stability and can be a factor in loan approvals.

Based on these attributes, our target variable is Loan Approved (Yes = 1, No = 0).

### Training Data

Here are the historical decisions your underwriters have made regarding loan approval and the
above-mentioned attributes.

| Applicant | Good Credit Score | Employment Status | Home Ownership | Existing Debts | Loan Approved |
| --------- | ----------------- | ----------------- | -------------- | -------------- | ------------- |
| 1         | 1                 | 1                 | 1              | 1              | 1             |
| 2         | 0                 | 1                 | 0              | 1              | 0             |
| 3         | 1                 | 0                 | 0              | 0              | 0             |
| 4         | 1                 | 1                 | 0              | 0              | 1             |
| 5         | 0                 | 0                 | 0              | 0              | 0             |

### Mathematical Definitions

There are a couple of mathematical definitions that we must establish before constructing a decision
tree.

#### Conditional Probability

Conditional probability is the probability of an event occurring given that another event has
already occurred. Mathematically, the conditional probability of an event
$B$ given an event $A$ is denoted by $P\left(B∣A\right)$, and it is defined as:

$$
  P(B∣A)= \dfrac{P(A)}{P(A \cap B)}
$$

where:
$P(A \cap B)$ is the probability that both events $A$ and $B$ occur,
$P(A)$ is the probability that event $A$ occurs.

#### Potential Function

A potential function is a measure of the uncertainty of a conditional probability represented within a decision tree.
If the potential function returns a 0 or 1, then we call the node _pure_; meaning that it directly
correlates to the label for all data in our sample $S$.

The Gini index is one such potential function.
We will use this to determine the next feature to select as the root
of the subtree as we construct the decision tree.

$$
  \phi (a) = 2a(1 − a).
$$

The Gini index was introduced by Breiman, Friedman, Olshen & Stone (1984) and has a helpful
property of being smooth and concave - which is often a helpful property in machine learning
applications.

### Decision Tree Algorithm Overview

To build a decision tree from this data, we will use a basic algorithmic approach
which involves the following steps:

- **Begin with a Trivial Root**: Select a root node whose value matches the majority of labels
  in the training data set and compute the error on this trivial tree.
  The initial error is defined as:

  $$
   \text{error}_{0} = \phi(\mathbb{P}_{x,y\sim S}[y=0])
  $$

  Note that $\mathbb{P}_{x,y\sim S}[y=0]$ reads as:

> The probability that when a random sample of data ${x}$ and its label ${y}$ drawn
> from the training set $S$ has a label of 0, meaning that the loan was not approved.

- **Choosing the Best Attribute using Information Gain**: Iterate over all potential
  attributes ${x_i}$ as the root of the next subtree and performing the following calculation:

  - **Calculate Error**:

  $$
  \begin{aligned}
  \text{error}_{x_i} =& \mathbb{P}_{x,y\sim S}[x_i=0] \cdot \phi(\mathbb{P}_{x,y\sim S}[y=0|x_i=0]) + \\
                    &\mathbb{P}_{x,y\sim S}[x_i=1] \cdot \phi(\mathbb{P}_{x,y\sim S}[y=0|x_i=1])
  \end{aligned}
  $$

  - **Calculate Gain**:

Take the difference of the prior error and the new error if ${x_i}$ was chosen as the root of the next subtree.

$$
\text{gain}_{x_i} = \text{error}_{x_{i-1}} - \text{error}_{x_i}
$$

- **Select Maximum**:

$$
\underset{x}{\mathrm{argmax}} \left(gain_{x_i}\right)
$$

- **Splitting the Data**: Based on the attribute selected, split the dataset into subsets
  that will then form the branches of the node.

- **Repeating the Process**: Repeat this process recursively for each branch, using
  the subsets of the dataset created by the split.

- **Stopping Criteria**: This can be when all attributes have been used, or when the
  subset at a node all have the same value of the target variable - this is called a _pure decision_.

### Underwriting Decision Tree

Now that we have an overview of the decision tree algorithm, let's compute a decision tree for the underwriting data set provided above.

We'll follow the decision tree algorithm steps outlined earlier to construct the decision tree.

### Step 1: Choosing the Root

The majority label in the training dataset indicates that most loans are not approved ($y=0$).

We will use this fact to construct the decision with a trivial root that has a value of zero.

<!-- prettier-ignore-start -->
{% plantuml %}
@startwbs
* 0

@endwbs
{% endplantuml %}
<!-- prettier-ignore-end -->

Thus, the initial error is computed as:

$$
\text{error}_{0} = \phi(\mathbb{P}_{x,y\sim S}[y=0])
\\
\text{error}_{0} = \phi(\dfrac{3}{5})
\\
\text{error}_{0} = 2 \cdot \dfrac{3}{5} \cdot \left(1 - \dfrac{3}{5} \right)
\\
\text{error}_{0} = \dfrac{12}{25}


$$

### Step 2: Find the next root.

#### If $x_i$ is "Good Credit Score"

| $\mathbb{P}_{x,y\sim S}[x_i=0]$ | $\mathbb{P}\_{x,y\sim S}[y=0 \| x_i=0]$ | $\mathbb{P}_{x,y\sim S}[x_i=1]$ | $\mathbb{P}\_{x,y\sim S}[y=0 \| x_i=1]$ |
| ------------------------------- | --------------------------------------- | ------------------------------- | --------------------------------------- |
| $\dfrac{2}{5}$                  | $\dfrac{2}{2}$                          | $\dfrac{3}{5}$                  | $\dfrac{1}{3}$                          |

So we would have that the error is defined as:

$$
 \text{error}_{x_i} = \dfrac{2}{5} \cdot \phi \left(\dfrac{2}{2}\right) + \dfrac{3}{5} \cdot \phi \left(\dfrac{1}{3}\right)

 \\
 \text{error}_{x_i} = \dfrac{2}{5} \cdot 0 + \dfrac{3}{5} \cdot \dfrac{4}{9}

 \\
 \text{error}_{x_i} = \dfrac{4}{15}
$$

And the gain would be defined as:

$$
  \text{gain}_{x_i} = \dfrac{12}{25} - \dfrac{4}{15}
  \\
  \text{gain}_{x_i} = \dfrac{16}{75}
$$

#### If $x_i$ is "Employment Status"

| $\mathbb{P}_{x,y\sim S}[x_i=0]$ | $\mathbb{P}\_{x,y\sim S}[y=0 \| x_i=0]$ | $\mathbb{P}_{x,y\sim S}[x_i=1]$ | $\mathbb{P}\_{x,y\sim S}[y=0 \| x_i=1]$ |
| ------------------------------- | --------------------------------------- | ------------------------------- | --------------------------------------- |
| $\dfrac{2}{5}$                  | $\dfrac{2}{2}$                          | $\dfrac{3}{5}$                  | $\dfrac{1}{3}$                          |

So we would have that the error is defined as:

$$
 \text{error}_{x_i} = \dfrac{2}{5} \cdot \phi \left(\dfrac{2}{2}\right) + \dfrac{3}{5} \cdot \phi \left(\dfrac{1}{3}\right)

 \\
 \text{error}_{x_i} = \dfrac{2}{5} \cdot 0 + \dfrac{3}{5} \cdot \dfrac{4}{9}

 \\
 \text{error}_{x_i} = \dfrac{4}{15}
$$

And the gain would be defined as:

$$
  \text{gain}_{x_i} = \dfrac{12}{25} - \dfrac{4}{15}
  \\
  \text{gain}_{x_i} = \dfrac{16}{75}
$$

#### If $x_i$ is "Home Ownership"

| $\mathbb{P}_{x,y\sim S}[x_i=0]$ | $\mathbb{P}\_{x,y\sim S}[y=0 \| x_i=0]$ | $\mathbb{P}_{x,y\sim S}[x_i=1]$ | $\mathbb{P}\_{x,y\sim S}[y=0 \| x_i=1]$ |
| ------------------------------- | --------------------------------------- | ------------------------------- | --------------------------------------- |
| $\dfrac{4}{5}$                  | $\dfrac{3}{4}$                          | $\dfrac{1}{5}$                  | $\dfrac{0}{1}$                          |

So we would have that the error is defined as:

$$
 \text{error}_{x_i} = \dfrac{4}{5} \cdot \phi \left(\dfrac{3}{4}\right) + \dfrac{1}{5} \cdot \phi \left(\dfrac{0}{1}\right)

 \\
 \text{error}_{x_i} = \dfrac{4}{5} \cdot \dfrac{3}{8} + \dfrac{1}{5} \cdot 0

 \\
 \text{error}_{x_i} = \dfrac{3}{10}
$$

And the gain would be defined as:

$$
  \text{gain}_{x_i} = \dfrac{12}{25} - \dfrac{3}{10}
  \\
  \text{gain}_{x_i} = \dfrac{9}{50}
$$

#### If $x_i$ is "Existing Debts"

| $\mathbb{P}_{x,y\sim S}[x_i=0]$ | $\mathbb{P}\_{x,y\sim S}[y=0 \| x_i=0]$ | $\mathbb{P}_{x,y\sim S}[x_i=1]$ | $\mathbb{P}\_{x,y\sim S}[y=0 \| x_i=1]$ |
| ------------------------------- | --------------------------------------- | ------------------------------- | --------------------------------------- |
| $\dfrac{3}{5}$                  | $\dfrac{2}{3}$                          | $\dfrac{2}{5}$                  | $\dfrac{1}{2}$                          |

So we would have that the error is defined as:

$$
 \text{error}_{x_i} = \dfrac{3}{5} \cdot \phi \left(\dfrac{2}{3}\right) + \dfrac{2}{5} \cdot \phi \left(\dfrac{1}{2}\right)

 \\
 \text{error}_{x_i} = \dfrac{3}{5} \cdot \dfrac{4}{9} + \dfrac{2}{5} \cdot \dfrac{1}{2}

 \\
 \text{error}_{x_i} = \dfrac{7}{15}
$$

And the gain would be defined as:

$$
  \text{gain}_{x_i} = \dfrac{12}{25} - \dfrac{2}{5}
  \\
  \text{gain}_{x_i} = \dfrac{2}{25}
$$

### Step 3: Select Maximum

Next we select the ${x_i}$ which gives us the most significant gain. The gain for each of the
potential candidate features is outlined below.

| Good Credit Score | Employment Status         | Home Ownership  | Existing Debts  |
| ----------------- | ------------------------- | --------------- | --------------- |
| $\dfrac{16}{75}$  | $\mathbf{\dfrac{16}{75}}$ | $\dfrac{9}{50}$ | $\dfrac{2}{25}$ |

"Good Credit Score" and "Employment Status" are tied for the maximum gain, so we will
choose "Employment Status" as the root of the tree.

<!-- prettier-ignore-start -->
{% plantuml %}
@startwbs
* Employment Status?

@endwbs
{% endplantuml %}
<!-- prettier-ignore-end -->

## Recurse

Now that we have a node for our decision tree, we simply recurse down and find out the next nodes using the conditional probability of the existing root.

First, we need to compute the error of this new tree. This is defined as:

$$
\begin{aligned}
  \text{error}_{x_i} =& \mathbb{P}_{x,y\sim S}[x_i=0] \cdot \phi(\mathbb{P}_{x,y\sim S}[y=0|x_i=0]) + \\
                    & \mathbb{P}_{x,y\sim S}[x_i=1] \cdot \phi(\mathbb{P}_{x,y\sim S}[y=0|x_i=1])\\
                    =& \dfrac{3}{10}
\end{aligned}
$$

**This act of partitioning the data based on the parent node is how decision trees are constructed.**

### Left Subtree

To find the root of the right subtree, we will examine the errors and gains produced by potential candidate features conditioned on "Employment Status" being 0.

| Applicant | Good Credit Score | Employment Status | Home Ownership | Existing Debts | Loan Approved |
| --------- | ----------------- | ----------------- | -------------- | -------------- | ------------- |
| 1         | 1                 | 1                 | 1              | 1              | 1             |
| 2         | 0                 | 1                 | 0              | 1              | 0             |
| **3**     | **1**             | **0**             | **0**          | **0**          | **0**         |
| 4         | 1                 | 1                 | 0              | 0              | 1             |
| **5**     | **0**             | **0**             | **0**          | **0**          | **0**         |

Note that in this situation, every row of data that has "Employment Status" of 0 results int he denial of their loan application.
This means that we have a pure decision; when the applicant is a home owner, we would predict they would receive loan approval.

<!-- prettier-ignore-start -->
{% plantuml %}
@startwbs
<style>
wbsDiagram {
  .green {
      BackgroundColor #90EE90
  }
  .red {
      BackgroundColor #FF474C
  }
  .gray {
      BackgroundColor #D3D3D3
  }
}
</style>
+ Employment Status?
++ DENY <<red>>
++ ... <<gray>>

@endwbs
{% endplantuml %}
<!-- prettier-ignore-end -->

### Right Subtree

To find the root of the right subtree, we will examine the errors and gains produced by potential candidate features conditioned on "Employment Status" being 1.

| Applicant | Good Credit Score | Employment Status | Home Ownership | Existing Debts | Loan Approved |
| --------- | ----------------- | ----------------- | -------------- | -------------- | ------------- |
| **1**     | **1**             | **1**             | **1**          | **1**          | **1**         |
| **2**     | **0**             | **1**             | **0**          | **1**          | **0**         |
| 3         | 1                 | 0                 | 0              | 0              | 0             |
| **4**     | **1**             | **1**             | **0**          | **0**          | **1**         |
| 5         | 0                 | 0                 | 0              | 0              | 0             |

We denote the probability of an event $E$ occurring when "Employment Status" is 1 as:

$$
  \mathbb{P}_{x,y\sim S_{|_{\text{Employment Status} = 1}}}[ E ]
$$

#### If $x_i$ is "Good Credit Score"

| $\mathbb{P}\_{x,y\sim S_{\|_{\text{Employment Status} = 0}}}[ x_i=0 ]$ | $\mathbb{P}\_{x,y\sim S_{\|_{\text{Employment Status} = 0}}}[y=0 \| x_i=0]$ | $\mathbb{P}\_{x,y\sim S_{\|_{\text{Employment Status} = 0}}}[ x_i=1 ]$ | $\mathbb{P}\_{x,y\sim S_{\|_{\text{Employment Status} = 0}}}[y=0 \| x_i=1]$ |
| ---------------------------------------------------------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| $\dfrac{1}{3}$                                                         | $\dfrac{1}{1}$                                                              | $\dfrac{2}{3}$                                                         | $\dfrac{0}{2}$                                                              |

So we would have that the error is defined as:

$$
 \text{error}_{x_i} = \dfrac{1}{3} \cdot \phi \left(\dfrac{1}{1}\right) + \dfrac{2}{3} \cdot \phi \left(\dfrac{0}{2}\right)

 \\
 \text{error}_{x_i} = \dfrac{1}{3} \cdot 0 + \dfrac{2}{3} \cdot 0

 \\
 \text{error}_{x_i} = 0
$$

And the gain would be defined as:

$$
  \text{gain}_{x_i} = \dfrac{4}{15} - 0
  \\
  \text{gain}_{x_i} = \dfrac{4}{15}
$$

Note that because we produce 0 error on "Good Credit Score" as a potential root of this tree, we are guaranteed that it is an optimal choice
for maximizing the gain, so there is no need to calculate the error and gain for other candidate roots.

At this point, we could recurse down yet another level, but the error on the "Good Credit Score" node is 0.
This again means that we have a _pure decision_; when the applicant is not employed, but they have a good credit score, we would predict they would receive loan approval, else, they would be denied.

<!-- prettier-ignore-start -->
{% plantuml %}
@startwbs
<style>
wbsDiagram {
  .green {
      BackgroundColor #90EE90
  }
  .red {
      BackgroundColor #FF474C
  }
}
</style>
+ Employment Status?
++ DENY <<red>>
++ Good Credit Score?
+++ APPROVE <<green>>
++- DENY <<red>>

@endwbs
{% endplantuml %}
<!-- prettier-ignore-end -->

### Accurate Score

The _accuracy score_ of a decision tree is simply the ratio of data points in $S$ that it labels
correctly over the total number of data points. For this tree we have an accuracy score of $\dfrac{5}{5}$.
This means that every example within our small training data set would be properly classified with this simple decision tree.

### Predictive Power

It's great that we have a model that fits all of the data it has seen, but how would we use this to classify a new loan application?

Let's say a new application comes in with the following attributes:

| Applicant | Good Credit Score | Employment Status | Home Ownership | Existing Debts |
| --------- | ----------------- | ----------------- | -------------- | -------------- |
| 6         | 1                 | 0                 | 1              | 1              |

Our decision tree would predict that this applicant would _not_ have their loan application approved.

## Encode it

You can see that the act of producing a decision tree is very simple, but the computations are rather repetitive.
Let's have a computer do most of this work for us. Below is a python script will produce a decision tree from this same
data set and use it to predict the outcome of the new loan application:

```py
import numpy as np
from sklearn.tree import DecisionTreeClassifier, export_text

# Define the training data and labels
X = np.array([
    [1, 1, 1, 1],
    [0, 1, 0, 1],
    [1, 0, 0, 0],
    [1, 1, 0, 0],
    [0, 0, 0, 0]
])

y = np.array([1, 0, 0, 1, 0])

# Create a decision tree classifier instance
# Criterion 'gini' refers to the Gini impurity
# max_depth is set to 2 to limit the depth of the tree
classifier = DecisionTreeClassifier(criterion='gini', max_depth=2)

# Fit the model to the data
classifier.fit(X, y)

# Print the tree as text
tree_rules = export_text(
  classifier,
  feature_names=[
    'Good Credit Score',
    'Employment Status',
    'Home Ownership',
    'Existing Debts'
  ]
)
print(tree_rules)
# |--- Employment Status <= 0.50
# |   |--- class: 0
# |--- Employment Status >  0.50
# |   |--- Good Credit Score <= 0.50
# |   |   |--- class: 0
# |   |--- Good Credit Score >  0.50
# |   |   |--- class: 1

# New loan application
new_data = np.array([[1, 0, 1, 1]])

# Predict the label for the new data point
predicted_label = classifier.predict(new_data)

print("Predicted label:", predicted_label[0])
# Predicted label: 0
```

## Conclusion

Decision trees are intuitive and widely used machine learning models that offer clear visual interpretations and straightforward decision-making pathways. Constructing a decision tree involves systematically splitting the data based on feature values that best separate the classes in the training dataset, aiming to achieve the purest possible subsets at each node.

In the context of loan approval, features such as credit score, employment status, existing debts, and home ownership play crucial roles in determining the likelihood of loan approval. By using metrics such as the Gini index, the model evaluates each feature's effectiveness at segregating the data into the two categories: loan approved (Yes = 1) and not approved (No = 0).

Through training on labeled data, decision trees thus act as binary classifiers that encapsulate and mimic human decision-making logic, making them highly valuable for applications like loan approval. The resulting model not only helps in predicting outcomes based on new data but also provides insights into the factors that most significantly impact those outcomes, making decision trees a powerful tool for both prediction and explanatory analysis.

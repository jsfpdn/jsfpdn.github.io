---
title: "Encoding Boolean Operators in Untyped λ Calculus"
date: 2023-04-07T10:13:00+02:00
tags: ["theory", "lambda calculus", "functional programming"]
math: true
summary: "Implementing boolean values and basic boolean operators in untyped λ calculus."
---

Untyped lambda calculus is about three things:

* **variables**, e.g. $x$
* **function abstraction** $\lambda x.M$ ($x$ is the argument of the function,
  $M$ is the function body which may or may not have occurances of $x$ and other variables),
* and **function application** $M N$ (applying argument $N$ to $M$).

In this post, I try to show and explain how boolean operators and pairs can be encoded
in untyped $\lambda$ calculus using only the three constructs.

> To follow along, it is useful to know what a <cite>reduction[^1] [^2]</cite> is.

## $\text{True}$ and $\text{False}$ terms

First, let's build $\text{True}$ and $\text{False}$ terms:

$$
\text{True} = \lambda xy. x \newline
\text{False} = \lambda xy. y
$$

$\text{True}$ and $\text{False}$ are nothing but a functions taking two arguments and return a single value.
$\text{True}$ returns the first argument $x$, $\text{False}$ returns the second argument $y$. That's it.

## If-then-else Operator

To move forward, we'll build an if-then-else operator, or $\text{ite}$ for short. It will be a function
taking three arguments -- a condition and two arguments to be evaluated depending on the result of the condition.

$$
\text{ite} \ \text{True} \ M \ N \rightarrow^* M \newline
\text{ite} \ \text{False} \ M \ N \rightarrow^* N
$$

> By $M \rightarrow^* N$ I mean that the term $M$ is $\beta \eta$-reduced to $N$ in zero or more steps.
> Also, I do not care about reduction strategies in this post, reduxes are chosen ad-hoc.

Let's dig into each case.

$$
\text{True} \ M \ N \rightarrow (\lambda xy. x) \ M \ N \rightarrow (\lambda y. M) \ N \rightarrow M \newline
\text{False} \ M \ N \rightarrow (\lambda xy. y) \ M \ N \rightarrow (\lambda y. y) \ N \rightarrow N
$$

$\text{True}$ and $\text{False}$ terms work as "selection operators" on their own! What we need
is to create the $\text{ite}$ function which will apply the condition to the two arguments and that's it.

$$
\text{ite} = \lambda xyz. xyz
$$

Let's check:

$$
\text{ite} \ \text{True} \ M \ N \newline
\rightarrow (\lambda xyz. xyz) \ (\lambda xy. x) \ M \ N \quad \small \text{// Substitute for ite and True} \newline
\rightarrow (\lambda yz. \ (\lambda xy. \ x)yz) \ M \ N  \quad \small \text{// Apply True to ite} \newline
\rightarrow (\lambda yz. \ (\lambda a. \ y)z) \ M \ N    \quad \small \text{// Reduce the body of ite} \newline
\rightarrow (\lambda yz. \ y) \ M \ N                    \quad \small \text{// Reduce the body of ite once again} \newline
\rightarrow (\lambda z. \ M) \ N                         \quad \small \text{// Apply function to M} \newline
\rightarrow M                                            \quad \small \text{// Apply function to N}
$$

## $\text{And}$ Operator

Let's build an $\text{And}$ operator. Intuitively, we aim to build something that acts like this:


$$
\text{And} \ \text{True} \ \text{True} \rightarrow^* \text{True} \newline
\text{And} \ \text{True} \ \text{False} \rightarrow^* \text{False} \newline
\text{And} \ \text{False} \ \text{True} \rightarrow^* \text{False} \newline
\text{And} \ \text{False} \ \text{False} \rightarrow^* \text{False}
$$

That is, we're building a function which takes two arguments and returns a single value $\text{True}$ or
$\text{False}$.
We can go further and case for each case plug in what we already know (that is $\text{True}$ and $\text{False}$ terms):

$$
\text{And} \ \text{True} \ \text{True} \rightarrow^* \text{And} \ (\lambda xy. x) \ (\lambda xy. x) \rightarrow^* (\lambda xy. x) \newline
\text{And} \ \text{True} \ \text{False} \rightarrow^* \text{And} \ (\lambda xy. x) \ (\lambda xy. y) \rightarrow^* (\lambda xy. y) \newline
\text{And} \ \text{False} \ \text{True} \rightarrow^* \text{And} \ (\lambda xy. y) \ (\lambda xy. x) \rightarrow^* (\lambda xy. y) \newline
\text{And} \ \text{False} \ \text{False} \rightarrow^* \text{And} \ (\lambda xy. y) \ (\lambda xy. y) \rightarrow^* (\lambda xy. y)
$$

We can look at it this way: if the first argument, $x$, is $\text{True}$, we must return the value of $y$.
Otherwise, we immediately know that the result of $\text{And}$ is $\text{False}$ (that is $x$).
This is the function that does exactly that:

$$
\text{And} = \lambda xy. \ \text{ite} \ x \ y \ x \newline
\rightarrow \lambda xy . \ (\lambda xyz. \ x \ y \ z) \ x \ y \ x \newline
\rightarrow \lambda xy . \ (\lambda yz. \ x \ y \ z) \ y \ x \newline
\rightarrow \lambda xy . \ (\lambda z. \ x \ y \ z) \ x \newline
\rightarrow \lambda xy . \ x \ y \ x
$$

Let's verify it on an example:

$$
\text{And} \ \text{True} \ \text{False} \newline
\rightarrow (\lambda xy. \ x \ y \ x) \ (\lambda xy. \ x) \ (\lambda xy . \ y)              \quad \small \text{// Substitute for And, True and False} \newline
\rightarrow (\lambda y. \ (\lambda xy . \ x) \ y \ (\lambda xy . \ x)) \ (\lambda xy . \ y) \quad \small \text{// Pass True to And} \newline
\rightarrow (\lambda xy. \ x) \ (\lambda xy . \ y) \ (\lambda xy . \ x)                         \quad \small \text{// Pass False to And} \newline
\rightarrow (\lambda y . \ (\lambda xy . \ y)) \ (\lambda xy . \ x)                         \quad \small \text{// Pass False to True} \newline
\rightarrow \lambda xy. \ y = \text{False}
$$

## $\text{Or}$ Operator

Similarily to $\text{And}$, $\text{Or}$ takes two terms. If the first argument $x$ is $\text{True}$, $\text{True}$ is returned,
otherwise $y$ is returned.

$$
\text{Or} = \lambda xy. \ \text{ite} \ x \ x \ y \newline
\rightarrow \lambda xy. \ (\lambda xyz . \ x \ y \ z) \ x \ x \ y \newline
\rightarrow \lambda xy. \ (\lambda yz . \ x \ y \ z) \ x \ y \newline
\rightarrow \lambda xy. \ (\lambda z . \ x \ x \ z) \ y \newline
\rightarrow \lambda xy. \ x \ x \ y \newline
$$

And an example:

$$
\text{Or} \ \text{False} \ \text{True} \newline
\rightarrow (\lambda xy. \ x \ x \ y) \ (\lambda xy. \ y) \ (\lambda xy. \ x) \newline
\rightarrow (\lambda y. (\lambda xy. \ y) \ (\lambda xy . \ y)) \ (\lambda xy . \ x) \newline
\rightarrow (\lambda xy. \ y) \ (\lambda xy. \ y) \ (\lambda xy. \ x) \newline
\rightarrow (\lambda y. \ y) \ (\lambda xy. \ x) \newline
\rightarrow \lambda xy. \ x = \text{True}
$$

## $\text{Not}$ Operator

$\text{Not}$ takes a single term $x$ and computes the negation.
If $x$ is $\text{True}$, $\text{False}$ is returned, otherwise $\text{True}$ is returned.

### Constructing $\text{Not}$ Intuitively

We'll begin with an intuitive construction of $\text{Not}$, building it exactly as we've described
above how it should behave:

$$
\text{Not} = \lambda x. \ \text{ite} \ x \ \text{False} \ \text{True} \newline
\rightarrow \lambda x. \ (\lambda xyz. \ x \ y \ z) \ x \ (\lambda xy. \ y) \ (\lambda xy. \ x) \newline
\rightarrow \lambda x. \ (\lambda yz. \ x \ y \ z) \ (\lambda xy. \ y) \ (\lambda xy . \ x) \newline
\rightarrow \lambda x. \ (\lambda z. \ x \ (\lambda xy. \ y) \ z) \ (\lambda xy . \ x) \newline
\rightarrow \lambda x. \ x \ (\lambda xy. \ y) \ (\lambda xy. \ x) \newline
\rightarrow \lambda x. \ x \ (\lambda y. \ y)
$$

The result does not seem so intuitive, so let's verify that it works:

$$
\text{Not} \ \text{True} \newline
\rightarrow (\lambda x. \ x \ (\lambda y. \ y)) (\lambda xy. \ x) \newline
\rightarrow (\lambda xy. \ x) \ (\lambda y. \ y) \newline
\rightarrow \lambda z. \ \lambda y. \ y \newline
\rightarrow \lambda zy. \ y \newline
\rightarrow_\alpha \lambda xy. \ y = \text{False}
$$

### Alternative $\text{Not}$ Construction

$\text{Not}$ can be alternatively defined as:

$$
\text{Not} = \lambda xyz. \ x \ z \ y
$$

Let's check $\text{Not} \ \text{False}$ this time:

$$
\text{Not} \ \text{False} \newline
\rightarrow (\lambda xyz. \ x \ z \ y) (\lambda xy. \ y) \newline
\rightarrow \lambda yz. \ (\lambda xy. \ y) \ z \ y \newline
\rightarrow \lambda yz. \ (\lambda y. \ y ) \ y \newline
\rightarrow \lambda yz. \ y \newline
\rightarrow_\alpha \lambda xy . \ x = \text{True}
$$

[^1]: [Wikipedia entry on λ-Calculus Reduction.](https://en.wikipedia.org/wiki/Lambda_calculus#Reduction)
[^2]: [Nice stackoverflow answer describing alpha and eta conversions and beta reduction.](https://stackoverflow.com/a/34305160/7843475)

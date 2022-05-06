---
layout: post
title: "Copulas in Probability Theory"
---

## Basic Concepts

The term "Copula" was first introduced by Abe Sklar in his 1959 article (see [^1]). It has been widely used, though not always applied properly, in financial and econometric modeling, especially in pricing securities that depend on many underlying securities.

<!-- excerpt-end -->

Properties of copulas are analogous to properties of joint distributions. The *joint distribution* of a set of random variables $(Y_{1}, \dots , Y_{m})$ is defined as

$$F(y_{1}, \dots , y_{m}) = \mathbb{P}\{Y_{i} \leq y_{i}; i = 1, \dots , m\},$$

and the **survival function** corresponding to $F(y_{1}, \dots , y_{m})$ is given by $$\overline{F}(y_{1}, \dots , y_{m}) = \mathbb{P}\{Y_{i} > y_{i}; i = 1, \dots , m\}.$$

**Definition 1.** An $n$-dimensional copula (briefly, an *n-copula*) is a function $C$ from the unit $n$-cube $[0, 1]^{n}$ to the unit interval $[0, 1]$ which satisfies the following conditions:

1. $C(1, \dots , 1, a_{m}, 1, \dots , 1) = a_{m}$ for each $m \leq n$ and all $a_{m} \in [0, 1]$.

2. $C(a_{1}, \dots , a_{n}) = 0$ if $a_{m} = 0$ for any $m \leq n$.

3. $C$ is $n$-increasing in the sense that the $C$-volume of any $n$-dimensional interval is non-negative. In particular, if $C$ is a $2$-dimensional copula, then

4. $C(a_{2}, b_{2}) - C(a_{1}, b_{2}) - C(a_{2}, b_{1}) + C(a_{1}, b_{1}) \geq 0$, whenever $a_{1} \leq a_{2}$, $b_{1} \leq b_{2}$.

**Property 1** says that if the realizations of $n - 1$ variables are known each with marginal probability one, then the joint probability of the $n$ outcomes is the same as the probability of the remaining uncertain outcome. **Property 2** (sometimes referred to as the **grounded** property of a copula) says that the joint probability of all outcomes is zero if the marginal probability of any outcome is zero. An $n$-copula may be viewed, or equivalently defined as an $n$-dimensional cumulative probability distribution function whose support is contained in $[0, 1]^{n}$ and whose one-dimensional margins are uniform on $[0, 1]$ (see [^2]). In other words, an $n$-copula is an $n$-dimensional distribution function with all $n$ univariate margins being $U(0, 1)$ (see [^3]).

Sklar's basic result is now the following:

**Theorem 1.** If $H$ is an $n$-dimensional probability distribution function with one-dimensional margins $F_{1}, \dots , F_{n}$, then there exists an $n$-dimensional copula $C$ such that, for all $x_{1}, \dots , x_{n} \in \mathbb{R}$, $$H(x_{1}, \dots , x_{n}) = C(F_{1}(x_{1}), \dots , F_{n}(x_{n})).$$

Moreover, letting $u_{i} = F_{i}(x_{i}), i = 1, \dots , n$, yields

$$C(u_{1}, \dots , u_{n}) = H(F^{-1}(u_{1}), \dots , F_{n}^{-1}(u_{n})),$$

where, for any one-dimensional distribution function $F$, $F^{-1}(t) = \sup\{x \mid F(x) < t\}$ (the "quantile transform"). These results show that much of the study of joint distribution functions can be reduced to the study of copulas!

To prove Sklar's theorem, we shall introduce the notion of "**distributional transform**":

**Definition 2.** Let $Y$ be a real random variable with distribution function $F$ and let $V$ be a random variable independent of $Y$, such that $V \sim U(0, 1)$. The modified distribution function $F(x, \lambda)$ is defined by

$$F(x, \lambda) := \mathbb{P}\{X < x\} + \lambda \mathbb{P}(Y = x).$$

We call $U := F(Y, V)$ the (generalized) distributional transform of $Y$.

**Proof of Theorem 1.**  Let $X = (X_{1}, \dots , X_{n})$ be a random vector on a probability space with distribution function $F$ and let $V \sim U(0, 1)$ be independent of $X$. Considering the distributional transforms $U_{i} := F_{i}(X_{i}, V), 1 \leq i \leq n$, we have $U_{i} \overset{d}{=} U(0, 1)$ and $X_{i} = F_{i}^{-1}(U_{i})$ a.s., $1 \leq i \leq n$. Thus defining $C$ to be the distribution function of $U = (U_{1}, \dots , U_{n})$ we obtain

$$
\begin{align}
F(x) &= \mathbb{P}\{X \leq x\} \\
     &= \mathbb{P}\{F_{i}^{-1}(U_{i}) \leq x_{i}, 1 \leq i \leq n\} \\
     &= \mathbb{P}\{U_{i} \leq F_{i}(x_{i}), 1 \leq i \leq n\} \\
     &= C(F_{1}(x_{1}), \dots , F_{n}(x_{n})),    
\end{align}
$$

i.e. $C$ is a copula of $F$.

## References

[^1]: Abe Sklar, Random Variables, Distribution Functions, and Copulas, *Kybernetika*, Vol. 9 (1973), No. 6, pp. 449-460.

[^2]: Berthold Schweizer, Thirty Years of Copulas, In: G. Dall' Aglio, S. Kotz, and G. Salinetti (eds.): *Advances in Probability Distributions with Given Marginals: Beyond the Copulas.* The Netherlands: Kluwer Academic Publishers.

[^3]: Pravin K. Trivedi and David M. Zimmer, *Copula Modeling: An Introduction for Practitioners* Foundations and Trends&reg in Econometrics, Vol. 1, No. 1 (2005) pp. 1-111.
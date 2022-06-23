+++
title = "Gaussian processes and Bayesian optimization"
description = "Optimization, sampling and gaussian processes, notes from the book 'Algorithms for optimization'"
date = 2020-04-15
+++

This should be pasted in [upmath](https://upmath.me) to get all the latex equations in SVG format for posting on svbtle blog. I use mathpix notes because it syncs all my snips and I can insert raw images served from Mathpix CDN (made from Mathpix notes).

Sources:

- [Algorithms for optimization](https://mitpress.mit.edu/books/algorithms-optimization) book
- [Pyro](https://pyro.ai/examples/bo.html)'s documentation

## Motivation behind these optimization techniques

For many optimization problems, function evaluations can be quite expensive. For example, deep learning parameters may require a week of GPU training. A common approach to build a _surrogate model_, which is a model of the optimization problem that can be efficiently optimized in lieu of the true objective function. Further evaluations of the true objective function can be used to improve the model. Fitting such models requires an initial set of points

## Sampling plans

These are sampling plans for covering the search space when we have limited resources.

### Full factorial

The _full factorial_ sampling plan places a grid of evenly spaced points over the search space. This approach is easy to implement, does not rely on randomness, and covers the space, but it uses a large number of points. Sampling grid is bounded as shown in the picture

![image](https://cdn.mathpix.com/snip/images/0vpLGVPhZB1DnjFeZUd7XPn03bpK17u0ZbIMdYpBuPM.original.fullsize.png)

**Exponentially increase design points when the dimensionality high.**

### Random sampling

Draw m random samples over the design space.

A uniform projection plan is a sampling plan over a discrete grid where the dis- tribution over each dimension is uniform.

![image](https://cdn.mathpix.com/snip/images/o_0tCEn2Rb6fSu52cE72G_lRFSC69D1wjyN1d1vE_M0.original.fullsize.png)

### Stratified sampling

An $m \times m$ grid could miss important information due to systematic regularities. Cells are sampled at a point chosen uni- formly at random from within the cell rather than at the cell’s center

![image](https://cdn.mathpix.com/snip/images/hchZo2Qlz8EyP5bTXzv3vrly-u_VYjYSio_LuyLoPw0.original.fullsize.png)

There are several other sampling plans. We skip for now.

## Surrogate models

Now we discuss how to use these samples to construct models of the objective function that can be used in place of the real objective function. These _surrogate_ models are inexpensive to calculate

### Fitting models

Suppose we have $m$ design points $X=\left\\{\mathbf{x}^{(1)},\mathbf{x}^{(2)},\ldots,\mathbf{x}^{(m)}\right\\}$ and function evaluations $\mathbf{y}=\left\\{y^{(1)}, y^{(2)}, \ldots, y^{(m)}\right\\}$, the model will predict

```latex
\hat{\mathbf{y}}=\left\{\hat{f}_{\mathbf{\theta}}\left(\mathbf{x}^{(1)}\right), \hat{f}_{\mathbf{\theta}}\left(\mathbf{x}^{(2)}\right), \ldots, \hat{f}_{\mathbf{\theta}}\left(\mathbf{x}^{(m)}\right)\right\}
```

A surrogate model $\hat{f}$ parameterized by $\theta$ is designed to mimic the true objective function $f$. The parameters $\theta$ can be adjusted to fit the model based on samples collected from $f$

```latex
\underset{\theta}{\operatorname{minimize}} \quad\|\mathbf{y}-\hat{\mathbf{y}}\|_{p}
```

This penalizes the deviation of the model only at the data points. There is no guarantee that the model will continue to fit well away from observed data, and model accuracy typically decreases the farther we go from the sampled points. This form of model fitting is called **regression**.

### Linear models

A simple surrogate model is the linear model, which has the form

$$\hat{f}=w_{0}+\mathbf{w}^{\top} \mathbf{x} \quad \boldsymbol{\theta}=\left\\{w_{0}, \mathbf{w}\right\\}$$

For an n-dimensional design space, the linear model has n + 1 parameters, and thus requires at least n + 1 samples to fit unambiguously.

$$\hat{f}=\boldsymbol{\theta}^{\top} \mathbf{x}$$

Finding an optimal θ requires solving a linear regression problem:

```latex
\operatorname{minimize}_{\Theta}\|\mathbf{y}-\mathbf{X} \boldsymbol{\theta}\|_{2}^{2}
```

where $X$ is a design matrix formed from $m$ data points

```latex
\mathbf{X}=\left[\begin{array}{c}
\left(\mathbf{x}^{(1)}\right)^{\top} \\
\left(\mathbf{x}^{(2)}\right)^{\top} \\
\vdots \\
\left(\mathbf{x}^{(m)}\right)^{\top}
\end{array}\right]
```

Linear regression has an analytic solution

$$\theta=X^{+} y$$

where $X^+$ is the Moore-Penrose pseudoinverse of $X$. In Julia, this is `pinv`

$$\mathbf{X}^{+}=\mathbf{X}^{\top}\left(\mathbf{X} \mathbf{X}^{\top}\right)^{-1}$$

### Basis functions

The linear model is a linear combination of the components of $x$:

$$\hat{f}(\mathbf{x})=\theta_{1} x_{1}+\cdots+\theta_{n} x_{n}=\sum_{i=1}^{n} \theta_{i} x_{i}=\theta^{\top} \mathbf{x}$$

which is a specific example of a more general linear combination of basis functions

$$\hat{f}(\mathbf{x})=\theta_{1} b_{1}(\mathbf{x})+\cdots+\theta_{q} b_{q}(\mathbf{x})=\sum_{i=1}^{q} \theta_{i} b_{i}(\mathbf{x})=\theta^{\top} \mathbf{b}(\mathbf{x})$$

This solves this problem

```latex
\operatorname{minimize}_{\mathbf{\theta}}\|\mathbf{y}-\mathbf{B} \theta\|_{2}^{2}
```

### Other methods

There are polynomial basis functions, sinosoidal and radial basis functions. All of these have their own curve fitting properties

### Fitting Noisy Objective Functions

Models fit using regression will pass as close as possible to every design point. When the objective function evaluations are noisy, complex models are likely to excessively contort themselves to pass through every point. However, smoother fits are often better predictors of the true underlying objective function.

We add _regularization_ (L2) term to linear models specified above.

```latex
\operatorname{minimize}_{\theta}\|\mathbf{y}-\mathbf{B} \theta\|_{2}^{2}+\lambda\|\mathbf{\theta}\|_{2}^{2}
```

### Which model to choose?

Generalization error can be estimated using techniques such as holdout, k-fold cross validation, and the bootstrap.

## Probablistic surrogate models

When using surrogate models for the purpose of optimization, it is often useful to quantify our confidence in the predictions of these models. One way to quantify our confidence is by taking a probabilistic approach to surrogate modeling. Most common one is **Guassian Process** which represents a distribution over functions.

We use Gaussian processes to infer a distribution over the values of different design points given the values of previously evaluated design points. We can incorporate gradient information and noisy measurements of the objective functions.

### Guassian distribution

n-dimensional is parameterized

$$\mathcal{N}(\mathbf{x} | \boldsymbol{\mu}, \mathbf{\Sigma})=(2 \pi)^{-n / 2}|\mathbf{\Sigma}|^{-1 / 2} \exp \left(-\frac{1}{2}(\mathbf{x}-\boldsymbol{\mu})^{\top} \boldsymbol{\Sigma}^{-1}(\mathbf{x}-\boldsymbol{\mu})\right)$$

Covariance matrices are always positive semidefinite. Sampled value is

```latex
\mathbf{x} \sim \mathcal{N}(\boldsymbol{\mu}, \mathbf{\Sigma})
```

Some examples

![image](https://cdn.mathpix.com/snip/images/bqFWltlkHv4WeYGf9lP0Ms4vLbmyhFxNdYidfbS6_n8.original.fullsize.png)

Two jointly Gaussian random variables a and b can be written

```latex
\left[\begin{array}{l}
\mathbf{a} \\
\mathbf{b}
\end{array}\right] \sim \mathcal{N}\left(\left[\begin{array}{l}
\boldsymbol{\mu}_{\mathbf{a}} \\
\boldsymbol{\mu}_{\mathbf{b}}
\end{array}\right],\left[\begin{array}{ll}
\mathbf{A} & \mathbf{C} \\
\mathbf{C}^{\top} & \mathbf{B}
\end{array}\right]\right)
```

We choose `MvNormal` because _marginal distribution_ for a vector of random variables is given by its corresponding mean and covariance

```latex
\mathbf{a} \sim \mathcal{N}\left(\boldsymbol{\mu}_{\mathrm{a}}, \mathbf{A}\right) \quad \mathbf{b} \sim \mathcal{N}\left(\boldsymbol{\mu}_{\mathbf{b}}, \mathbf{B}\right)
```

and conditional distribution has a convenient closed form

```latex
\begin{array}{l}
\mathbf{a} | \mathbf{b} \sim \mathcal{N}\left(\mu_{\mathrm{a} | \mathbf{b}}, \mathbf{\Sigma}_{\mathrm{a} | \mathbf{b}}\right) \\
\boldsymbol{\mu}_{\mathrm{a} | \mathbf{b}}=\boldsymbol{\mu}_{\mathrm{a}}+\mathbf{C B}^{-1}\left(\mathbf{b}-\boldsymbol{\mu}_{\mathbf{b}}\right) \\
\mathbf{\Sigma}_{\mathrm{a} | \mathbf{b}}=\mathbf{A}-\mathbf{C B}^{-1} \mathbf{C}^{\top}
\end{array}
```

Simple example

```latex
\left[\begin{array}{l}
x_{1} \\
x_{2}
\end{array}\right] \sim \mathcal{N}\left(\left[\begin{array}{l}
0 \\
1
\end{array}\right],\left[\begin{array}{ll}
3 & 1 \\
1 & 2
\end{array}\right]\right)
```

The marginal distribution for $x_1$ is N (0, 3), and the marginal distribution for $x_2$ is $N (1, 2)$.

The conditional distribution for $x_1$ given $x_2 = 2$ is

```latex
\begin{aligned}
\boldsymbol{\mu}_{x_{1} | x_{2}=2} &=0+1 \cdot 2^{-1} \cdot(2-1)=0.5 \\
\boldsymbol{\Sigma}_{x_{1} | x_{2}=2} &=3-1 \cdot 1^{-1} \cdot 1=2.5 \\
x_{1} |\left(x_{2}=2\right) & \sim \mathcal{N}(0.5,2.5)
\end{aligned}
```

### Guassian processes

we approximated the objective function f using a surrogate model function $f^{hat}$ fitted to previously evaluated design points. A special type of surrogate model known as a Gaussian process allows us not only to predict $f$ but also to quantify our uncertainty in that prediction using a probability distribution

A Gaussian process is a distribution over functions. For any finite set of points $\left\\{\mathbf{x}^{(1)}, \ldots, \mathbf{x}^{(m)}\right\\}$, the associated function evaluations $\left\\{y_{1}, \ldots, y_{m}\right\\}$ are distributed according to:

```latex
\left[\begin{array}{c}
y_{1} \\
\vdots \\
y_{m}
\end{array}\right] \sim \mathcal{N}\left(\left[\begin{array}{c}
m\left(\mathbf{x}^{(1)}\right) \\
\vdots \\
m\left(\mathbf{x}^{(m)}\right)
\end{array}\right],\left[\begin{array}{ccc}
k\left(\mathbf{x}^{(1)}, \mathbf{x}^{(1)}\right) & \cdots & k\left(\mathbf{x}^{(1)}, \mathbf{x}^{(m)}\right) \\
\vdots & \ddots & \vdots \\
k\left(\mathbf{x}^{(m)}, \mathbf{x}^{(1)}\right) & \cdots & k\left(\mathbf{x}^{(m)}, \mathbf{x}^{(m)}\right)
\end{array}\right]\right)
```

where m(x) ($m(\mathbf{x})=\mathbb{E}[f(\mathbf{x})]$ ) is _mean function_ and k(x, x') ( $k\left(\mathbf{x}, \mathbf{x}^{\prime}\right)=$
$\mathbb{E}\left[(f(\mathbf{x})-m(\mathbf{x}))\left(f\left(\mathbf{x}^{\prime}\right)-m\left(\mathbf{x}^{\prime}\right)\right)\right]$ ) is _covariance function_ or _**kernel**_. The mean function can represent prior knowledge about the function. The kernel controls the smoothness of the functions.

Example psuedo code

![image](https://cdn.mathpix.com/snip/images/xknDFDS7v7rpHb8GLvDsCm_Z3BzWYulKP4MnYryx1mw.original.fullsize.png)

Common kernel function is squared exponential. There are several others, usually they use `r` which is euclidean distance between x and x'. Matern Kernel uses gamma function and and Kν(x) is the modified Bessel function of the second kind, for example

```latex
\frac{2^{1-v}}{\Gamma(v)}\left(\sqrt{2 v} \frac{r}{\ell}\right)^{v} K_{v}\left(\sqrt{2 v} \frac{r}{\ell}\right)
```

### Prediction

Suppose we already have a set of points X and the corresponding y, but we wish to predict the values yˆ at points $X^{*}$. The joint distribution is

```latex
\left[\begin{array}{l}
\hat{\mathbf{y}} \\
\mathbf{y}
\end{array}\right] \sim \mathcal{N}\left(\left[\begin{array}{l}
\mathbf{m}\left(X^{*}\right) \\
\mathbf{m}(X)
\end{array}\right],\left[\begin{array}{ll}
\mathbf{K}\left(X^{*}, X^{*}\right) & \mathbf{K}\left(X^{*}, X\right) \\
\mathbf{K}\left(X, X^{*}\right) & \mathbf{K}(X, X)
\end{array}\right]\right)
```

where m and k are

```latex
\begin{aligned}
\mathbf{m}(X) &=\left[m\left(\mathbf{x}^{(1)}\right), \ldots, m\left(\mathbf{x}^{(n)}\right)\right] \\
\mathbf{K}\left(X, X^{\prime}\right) &=\left[\begin{array}{ccc}
k\left(\mathbf{x}^{(1)}, \mathbf{x}^{\prime}(1)\right. & \cdots & k\left(\mathbf{x}^{(1)}, \mathbf{x}^{\prime(m)}\right) \\
\vdots & \ddots & \vdots \\
k\left(\mathbf{x}^{(n)}, \mathbf{x}^{\prime(1)}\right) & \cdots & k\left(\mathbf{x}^{(n)}, \mathbf{x}^{\prime(m)}\right)
\end{array}\right]
\end{aligned}
```

Therefore the conditional distribution is given by,

```latex
\hat{\mathbf{y}} | \mathbf{y} \sim \mathcal{N}(\underbrace{\mathbf{m}\left(X^{*}\right)+\mathbf{K}\left(X^{*}, X\right) \mathbf{K}(X, X)^{-1}(\mathbf{y}-\mathbf{m}(X))}_{\text {mean }}, \underbrace{\mathbf{K}\left(X^{*}, X^{*}\right)-\mathbf{K}\left(X^{*}, X\right) \mathbf{K}(X, X)^{-1} \mathbf{K}\left(X, X^{*}\right)}_{\text {covariance }})
```

Note, that the covariance is not dependent on y. This distribution is often referred to as the posterior distribution. In julia, `mvnrand(μ(X, GP.m), Σ(X, GP.k))`. The predicted mean

```latex
\begin{aligned}
\hat{\mu}(\mathbf{x}) &=m(\mathbf{x})+\mathbf{K}(\mathbf{x}, X) \mathbf{K}(X, X)^{-1}(\mathbf{y}-\mathbf{m}(X)) \\
&=m(\mathbf{x})+\boldsymbol{\theta}^{\top} \mathbf{K}(X, \mathbf{x})
\end{aligned}
```

where $\boldsymbol{\theta}=\mathbf{K}(X, X)^{-1}(\mathbf{y}-\mathbf{m}(X))$

_The value of the Gaussian process beyond the surrogate models discussed previously is that it also quantifies our uncertainty in our predictions._

The variance of the predicted mean can also be obtained as a function of x: $\hat{v}(\mathbf{x})=\mathbf{K}(\mathbf{x}, \mathbf{x})-\mathbf{K}(\mathbf{x}, X) \mathbf{K}(X, X)^{-1} \mathbf{K}(X, \mathbf{x})$

### Incorporating gradient measurements

Gaussian processes can be extended to incorporate gradients

```latex
\left[\begin{array}{c}
\mathbf{y} \\
\nabla \mathbf{y}
\end{array}\right] \sim \mathcal{N}\left(\left[\begin{array}{c}
\mathbf{m}_{f} \\
\mathbf{m}_{\nabla}
\end{array}\right],\left[\begin{array}{cc}
\mathbf{K}_{f f} & \mathbf{K}_{f \nabla} \\
\mathbf{K}_{\nabla f} & \mathbf{K}_{\nabla \nabla}
\end{array}\right]\right)
```

where $\mathbf{y} \sim \mathcal{N}\left(\mathbf{m}\_{f}, \mathbf{K}\_{f f}\right)$ is a traditional guassian process, $m_{\nabla}$ is a mean function of the gradient, $K_{\nabla f}$ is covariance matrix between function values and gradients etc.

The linearity of Gaussians causes these covariance functions to be related

$\begin{aligned} k_{f f}\left(\mathbf{x}, \mathbf{x}^{\prime}\right) &=k\left(\mathbf{x}, \mathbf{x}^{\prime}\right) \\ k_{\nabla f}\left(\mathbf{x}, \mathbf{x}^{\prime}\right) &=\nabla_{\mathbf{x}} k\left(\mathbf{x}, \mathbf{x}^{\prime}\right) \\ k_{f \nabla}\left(\mathbf{x}, \mathbf{x}^{\prime}\right) &=\nabla_{\mathbf{x}^{\prime}} k\left(\mathbf{x}, \mathbf{x}^{\prime}\right) \\ k_{\nabla \nabla}\left(\mathbf{x}, \mathbf{x}^{\prime}\right) &=\nabla_{\mathbf{x}} \nabla_{\mathbf{x}^{\prime}} k\left(\mathbf{x}, \mathbf{x}^{\prime}\right) \end{aligned}$

Prediction can be accomplished in the same manner as with a traditional Gaussian process. We first construct the joint distribution

```latex
\left[\begin{array}{c}
\hat{\mathbf{y}} \\
\mathbf{y} \\
\nabla \mathbf{y}
\end{array}\right] \sim \mathcal{N}\left(\left[\begin{array}{c}
\mathbf{m}_{f}\left(X^{*}\right) \\
\mathbf{m}_{f}(X) \\
\mathbf{m}_{\nabla}(X)
\end{array}\right],\left[\begin{array}{ccc}
\mathbf{K}_{f f}\left(X^{*}, X^{*}\right) & \mathbf{K}_{f f}\left(X^{*}, X\right) & \mathbf{K}_{f \nabla}\left(X^{*}, X\right) \\
\mathbf{K}_{f f}\left(X, X^{*}\right) & \mathbf{K}_{f f}(X, X) & \mathbf{K}_{f \nabla}(X, X) \\
\mathbf{K}_{\nabla f}\left(X, X^{*}\right) & \mathbf{K}_{\nabla f}(X, X) & \mathbf{K}_{\nabla \nabla}(X, X)
\end{array}\right]\right)
```

The conditional distribution follows the same Gaussian relations as in equation

```latex
\hat{\mathbf{y}} | \mathbf{y}, \nabla \mathbf{y} \sim \mathcal{N}\left(\boldsymbol{\mu}_{\nabla}, \mathbf{\Sigma}_{\nabla}\right)
```

```latex
\begin{aligned}
&\boldsymbol{\mu}_{\nabla}=\mathbf{m}_{f}\left(X^{*}\right)+\left[\begin{array}{c}
\mathbf{K}_{f f}\left(X, X^{*}\right) \\
\mathbf{K}_{\nabla f}\left(X, X^{*}\right)
\end{array}\right]^{\top}\left[\begin{array}{cc}
\mathbf{K}_{f f}(X, X) & \mathbf{K}_{f \nabla}(X, X) \\
\mathbf{K}_{\nabla f}(X, X) & \mathbf{K}_{\nabla \nabla}(X, X)
\end{array}\right]^{-1}\left[\begin{array}{c}
\mathbf{y}-\mathbf{m}_{f}(X) \\
\nabla \mathbf{y}-\mathbf{m}_{\nabla}(X)
\end{array}\right]\\
&\boldsymbol{\Sigma}_{\nabla}=\mathbf{K}_{f f}\left(X^{*}, X^{*}\right)-\left[\begin{array}{cc}
\mathbf{K}_{f f}\left(X, X^{*}\right) \\
\mathbf{K}_{\nabla f}\left(X, X^{*}\right)
\end{array}\right]^{\top}\left[\begin{array}{cc}
\mathbf{K}_{f f}(X, X) & \mathbf{K}_{f \nabla}(X, X) \\
\mathbf{K}_{\nabla f}(X, X) & \mathbf{K}_{\nabla \nabla}(X, X)
\end{array}\right]^{-1}\left[\begin{array}{c}
\mathbf{K}_{f f}\left(X, X^{*}\right) \\
\mathbf{K}_{\nabla f}\left(X, X^{*}\right)
\end{array}\right]
\end{aligned}
```

### Incorporating noisy measurements

Joint distribution

```latex
\left[\begin{array}{l}
\hat{\mathbf{y}} \\
\mathbf{y}
\end{array}\right] \sim \mathcal{N}\left(\left[\begin{array}{l}
\mathbf{m}\left(X^{*}\right) \\
\mathbf{m}(X)
\end{array}\right],\left[\begin{array}{ll}
\mathbf{K}\left(X^{*}, X^{*}\right) & \mathbf{K}\left(X^{*}, X\right) \\
\mathbf{K}\left(X, X^{*}\right) & \mathbf{K}(X, X)+v \mathbf{I}
\end{array}\right]\right)
```

with conditional distribution

```latex
\begin{aligned}
\hat{\mathbf{y}} | \mathbf{y}, \nu & \sim \mathcal{N}\left(\boldsymbol{\mu}^{*}, \mathbf{\Sigma}^{*}\right) \\
\boldsymbol{\mu}^{*} &=\mathbf{m}\left(X^{*}\right)+\mathbf{K}\left(X^{*}, X\right)(\mathbf{K}(X, X)+v \mathbf{I})^{-1}(\mathbf{y}-\mathbf{m}(X)) \\
\boldsymbol{\Sigma}^{*} &=\mathbf{K}\left(X^{*}, X^{*}\right)-\mathbf{K}\left(X^{*}, X\right)(\mathbf{K}(X, X)+v \mathbf{I})^{-1} \mathbf{K}\left(X, X^{*}\right)
\end{aligned}
```

### Fitting guassian processes

The choice of kernel and parameters has a large effect on the form of the Gaussian process between evaluated design points.

Kernels and their parameters can be chosen using cross validation introduced in the previous chapter. Instead of minimizing the squared error on the test data, we maximize the likelihood of the data.

> Likelihood math is hand-written in Cross Entropy method

We want the parameters $\theta$ that maximizes $p(\mathbf{y} | X, \boldsymbol{\theta})$. The likelihood of the data is the probability that the observed points were drawn from the model.

We use log likelihood,

```latex
\log p(\mathbf{y} | X, v, \mathbf{\theta})=-\frac{n}{2} \log 2 \pi-\frac{1}{2} \log \left|\mathbf{K}_{\mathbf{\theta}}(X, X)+v \mathbf{I}\right|-\frac{1}{2}\left(\mathbf{y}-\mathbf{m}_{\mathbf{\theta}}(X)\right)^{\top}\left(\mathbf{K}_{\mathbf{\theta}}(X, X)+v \mathbf{I}\right)^{-1}\left(\mathbf{y}-\mathbf{m}_{\mathbf{\theta}}(X)\right)
```

Let us assume a zero mean such that $\mathbf{m}_{\theta}(X)=\mathbf{0}$ and $\theta$ refers only to the parameters for the Gaussian process covariance function. We can arrive at a maximum likelihood estimate by gradient ascent.

```latex
\frac{\partial}{\partial \theta_{j}} \log p(\mathbf{y} | X, \theta)=\frac{1}{2} \mathbf{y}^{\top} \mathbf{K}^{-1} \frac{\partial \mathbf{K}}{\partial \theta_{j}} \mathbf{K}^{-1} \mathbf{y}-\frac{1}{2} \operatorname{tr}\left(\mathbf{\Sigma}_{\mathbf{\theta}}^{-1} \frac{\partial \mathbf{K}}{\partial \theta_{j}}\right)
```

```latex
\mathbf{\Sigma}_{\mathbf{\theta}}=\mathbf{K}_{\mathbf{\theta}}(X, X)+v \mathbf{I}
```

Results:

```latex
\begin{aligned}
\frac{\partial \mathbf{K}^{-1}}{\partial \boldsymbol{\theta}_{j}} &=-\mathbf{K}^{-1} \frac{\partial \mathbf{K}}{\partial \boldsymbol{\theta}_{j}} \mathbf{K}^{-1} \\
\frac{\partial \log |\mathbf{K}|}{\partial \boldsymbol{\theta}_{j}} &=\operatorname{tr}\left(\mathbf{K}^{-1} \frac{\partial \mathbf{K}}{\partial \boldsymbol{\theta}_{j}}\right)
\end{aligned}
```

## Optimization using Guassian process optimization

The last section discusses how to predict at a new point. We derived equations for estimating yhat at any new point. But how do we know which point to evaluate?
We want to move to a design point that minimizes our objective function.

### Prediction based exploration

In prediction-based exploration, we select the minimizer of the surrogate function.
An example of this approach is the quadratic fit. With quadratic fit search, we use a quadratic surrogate model to fit the last three bracketing points and then select the point at the minimum of the quadratic function.

If we use a Gaussian process surrogate model, prediction-based optimization has us select the minimizer of the mean function

```latex
\begin{aligned}
&\mathbf{x}^{(m+1)}=\arg \min _{x \in \mathcal{X}} \hat{\mu}(\mathbf{x})
\end{aligned}
```

where $\hat{\mu}(\mathbf{x})$ is the predicted mean of a Gaussian process at a design point x based on the previous m design points. This is not efficient. Prediction-based optimization does not take uncertainty into account, and new samples can be generated very close to existing samples rendering function evalutions useless.

### Error based exploration

Error-based exploration seeks to increase confidence in the true function. A Gaussian process can tell us both the mean and standard deviation at every point. A large standard deviation indicates low confidence, so error-based exploration samples at design points with maximum uncertainty.

The next sample point

```latex
\begin{aligned}
&\mathbf{x}^{(m+1)}=\arg \min _{x \in \mathcal{X}} \hat{\sigma}(\mathbf{x})
\end{aligned}
```

Optimization problems with unbounded feasible sets will always have high uncertainty far away from sampled points, making it impossible to become confident in the true underlying function over the entire domain

### Lower confidence bound exploration

Lower confidence bound exploration trades off between greedy minimization employed by prediction-based optimization and uncertainty reduction employed by error-based exploration. The next sample minimizes the lower confidence bound of the objective function

```latex
L B(\mathbf{x})=\hat{\mu}(\mathbf{x})-\alpha \hat{\sigma}(\mathbf{x})
```

where α ≥ 0 is a constant that controls the trade-off between exploration and exploitation. Exploration involves minimizing uncertainty, and exploitation involves minimizing the predicted mean

![image](https://cdn.mathpix.com/snip/images/hTnw50c9dDwDJHh7c_Y4LTBuh6zdA1s5Y2jVxcqj17g.original.fullsize.png)

## Summary

In essence, we have initial y, X for original objective function. We approximate using Guassian process using the formula

```latex
\left[\begin{array}{c}
y_{1} \\
\vdots \\
y_{m}
\end{array}\right] \sim \mathcal{N}\left(\left[\begin{array}{c}
m\left(\mathbf{x}^{(1)}\right) \\
\vdots \\
m\left(\mathbf{x}^{(m)}\right)
\end{array}\right],\left[\begin{array}{ccc}
k\left(\mathbf{x}^{(1)}, \mathbf{x}^{(1)}\right) & \cdots & k\left(\mathbf{x}^{(1)}, \mathbf{x}^{(m)}\right) \\
\vdots & \ddots & \vdots \\
k\left(\mathbf{x}^{(m)}, \mathbf{x}^{(1)}\right) & \cdots & k\left(\mathbf{x}^{(m)}, \mathbf{x}^{(m)}\right)
\end{array}\right]\right)
```

We get next sample design point and evaluate/update our gaussian process model (x, y) using yhat formula above. Then we get another design point, evaluate/update our surrogate model.

Remember the equation shown below has $m(x^{\*})$ which is $E[f(x^{\*})]$ which requires main objective function evaluation for updating posterior.

> Hence, updating posterior requires finding new design point and updating our gaussian process model. We use xnew and it's evaluation on out main objective function. Once we make our gaussian process model more robust after adequate iterations, our new design point is our minimum. For sampling new design point, refer to Lower confidence bound exploration.

Taken from BO link.

```python
def update_posterior(x_new):
    y = f(x_new) # evaluate f at new point.
    X = torch.cat([gpmodel.X, x_new]) # incorporate new evaluation
    y = torch.cat([gpmodel.y, y])
    gpmodel.set_data(X, y)
    # optimize the GP hyperparameters using Adam with lr=0.001
    optimizer = torch.optim.Adam(gpmodel.parameters(), lr=0.001)
    gp.util.train(gpmodel, optimizer)

```

same as

```latex
\hat{\mathbf{y}} | \mathbf{y} \sim \mathcal{N}(\underbrace{\mathbf{m}\left(X^{*}\right)+\mathbf{K}\left(X^{*}, X\right) \mathbf{K}(X, X)^{-1}(\mathbf{y}-\mathbf{m}(X))}_{\text {mean }}, \underbrace{\mathbf{K}\left(X^{*}, X^{*}\right)-\mathbf{K}\left(X^{*}, X\right) \mathbf{K}(X, X)^{-1} \mathbf{K}\left(X, X^{*}\right)}_{\text {covariance }})
```

Note that the covariance does not depend on y. This distribution is often referred to as the posterior distribution.

After doing this for a multiple points, we finally converge to our global minimum on our main objective function.

[Bayesian optimization in Pyro](https://pyro.ai/examples/bo.html) discusses the code for this. It's very helpful.

The Bayesian optimization strategy works as follows:

- Place a prior on the objective function $f$

- Each time we evaluate $f$ at a new point $x_{n}$, we update our model for $f(x)$. This model serves as a surrogate objective function and reflects our beliefs about $f$ (in particular it reflects our beliefs about where we expect $f(x)$ to be close to $f(x^{*})$

- Since we are being Bayesian, our beliefs are encoded in a posterior that allows us to systematically reason about the uncertainty of our model predictions.

- Use the posterior to derive an “acquisition” (prediction based, error based or lower bound) function $\alpha(x)$ that is easy to evaluate and differentiate (so that optimizing $\alpha(x)$ is easy). In contrast to $f(x)$, we will generally evaluate $\alpha(x)$ at many points $x$, since doing so will be cheap.

- Repeat until convergence

Next design point:

- Use the acquisition function to derive the next query point according to $x_{n+1}=\arg \min \alpha(x)$

- Evaluate $f(x_{n+1})$ and update the posterior.

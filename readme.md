# My Regents Scholar Research Initiative Project

I am working with Professor Arya Mazumdar to study certain behaviors of logistic regression. Below is the documentation of my experiments and process.

# Logistic Regressions

## Motivation

On the sample complexity of estimation in logistic regression | Daniel Hsu, Arya Mazumdar

> The logistic regression model is one of the most popular data generation model in noisy binary classification problems. In this work, we study the sample complexity of estimating the parameters of the logistic regression model up to a given ℓ2 error, in terms of the dimension and the inverse temperature, with standard normal covariates. The inverse temperature controls the signal-to-noise ratio of the data generation process. While both generalization bounds and asymptotic performance of the maximum-likelihood estimator for logistic regression are well-studied, the non-asymptotic sample complexity that shows the dependence on error and the inverse temperature for parameter estimation is absent from previous analyses. We show that the sample complexity curve has two change-points (or critical points) in terms of the inverse temperature, clearly separating the low, moderate, and high temperature regimes.

[Link to the paper here.](https://arxiv.org/pdf/2307.04191.pdf)

## Mathematical Background

A summary of the above paper is presented as follows. The logistic regression model is one with continuous input $X \in \mathbb{R}$ and outputs a binary classifier $y \in \{0, 1\}$ (sometimes defined as {-1, +1} instead):

$$y = \begin{cases}
    1 & \text{ with prob. } \dfrac{1}{1+\exp(-\beta\langle X, \theta \rangle)} \\
    0 & \text{ with prob. } \dfrac{1}{1+\exp(\beta\langle X, \theta \rangle)}
\end{cases}$$

where $X = \{x_1, x_2, x_3, ..., x_n\}$, where $x_i \in \mathbb{R}^d$ and $\theta \in \mathbb{R}^d, ||\theta|| = 1.$

The value $\beta$, considered the inverse temperature, is what governs the \textit{signal-to-noise ratio}; when $\beta = 0$, we have pure noise; when $\beta = \infty$, we simply have a linear classifier which denotes where $x$ lies on the hyperplane. Then, in this project, we seek to validate the paper above's claim that the sample complexity $n^*(d, \beta, \epsilon)$, which is the smallest sample size $n$ such that $||\theta - \hat{\theta}|| < \epsilon$, satisfies

$$n^*(d, \beta, \epsilon) \asymp \begin{cases}
    \dfrac{d}{\beta^2\epsilon^2} & \text{ if } \beta ≲ 1 \text{ (high temperatures);} \\
    \dfrac{d}{\beta\epsilon^2} & \text{ if } 1 ≲ \beta ≲ \dfrac{1}{\epsilon} \text{ (medium temperatures);} \\
    \dfrac{d}{\beta^2\epsilon^2} & \text{ if } \beta ≳ \dfrac{1}{\epsilon} \text{ (low temperatures);}
\end{cases}$$

## Data Generation

To generate the dataset, I generated $\theta \sim N(0, 1)$ and $X \sim U[-1, 1]$, producing an output binary classification label $y = \text{Bern}(\sigma(X^T \cdot \theta))$ where $\sigma(\eta)$ is the sigmoid function $(1+\exp(-\eta))^{-1}$.
```python
# Dataset generation
#   Generate an array of floats from 0 to 1, and dot with theta to find z
#   Probabilities result from applying sigmoid to the z array
#   Theta is drawn from N(0,1) and normalized to a given value of beta
#   X is drawn randomly from U[-1, 1]
def generate_data(beta, n):
    np.random.seed(SEED)
    theta_gen = np.random.normal(size=(NUM_FEATURES, 1))
    theta_gen = theta_gen * beta / np.linalg.norm(theta_gen)
    X_gen = np.random.rand(n, NUM_FEATURES) * 2 - 1
    z_gen = np.dot(X_gen, theta_gen)
    y_gen = np.random.binomial(1, sigmoid(z_gen))
    return theta_gen, X_gen, y_gen
```

## Basic Scikit-Learn Model

First, I started out with a basic sci-kit model estimating a randomly generated $\theta$ when varying $n$ (sample size) and $||\theta||$ (norm of the parameter vector). I used the `LogisticRegression` class from `sklearn.linear_model` to produce various graphs with the RMSE. The sample code can be found in the folder. This model proved to not be enough as the plain setup yielded results $\hat{\theta}$ which did not approach the true norm of the parameter vector. Instead, we chose to move to [CVXPY](https://www.cvxpy.org/index.html) (convex optimization programming library).

## CVXPY

CVXPY is a library which simplifies convex optimization problems (in our case, minimizing negative logistic loss). The setup for the problem is relatively simple:
```python
# Initialize the CVXPY maximization problem.
def setup(self):
    self.weights = cp.Variable(NUM_FEATURES)
    log_likelihood = cp.sum(
        cp.multiply(np.ndarray.flatten(self.y), self.X @ self.weights) - cp.logistic(self.X @ self.weights)
    )
    constraints = [cp.norm(self.weights) <= self.beta]
    return cp.Problem(cp.Maximize(log_likelihood/self.n), constraints)
```

## Results

An initial eye estimate of critical points $n^*$ based on plotting RMSE vs $n$ averaged over 5-10 runs of each $\beta$.
| $\beta$ | $\epsilon$ | $n^*$  |
|---------|------------|--------|
| 1.0     | 1.0        | 460    |
| 2.0     | 0.5        | 1308   |
| 3.0     | 0.333      | 1759   |
| 4.0     | 0.25       | 2125   |
| 5.0     | 0.2        | 2578   |
| 6.0     | 0.166      | 2994   |
| 7.0     | 0.143      | 3404   |
| 8.0     | 0.125      | 3808   |
| 9.0     | 0.111      | 4220   |
| 10.0    | 0.1        | 4690   |
| 20.0    | 0.05       | 9350   |
| 50.0    | 0.02       | ~21000 |

I took numbers based on the graph shown here:

<img src="Images/Figure_1.png" width="100%">

For $\beta = 1.0$, then, I averaged 1000 simulations per every $\Delta n=5$ centered around $n=500$, and found that it was closer to 460. Then I ran 1000 more simulations centered around $n=460$, and narrowed it down to $n^* \in [455, 465]$ and produced 3000 simulations per value of $n$ with $\Delta n = 1$, and the graph still wasn't smoothing out.

<img src="Images/Figure_2.png" width="100%">

Instead, I produced a best fit line with the points $n=450, 455, 456, 457, ..., 465, 470$, assuming that the curve was approximately linear at this small scale. Thus, $n^* \approx 460$ for $\beta = 1$. I repeat this with different scales for other values of $\beta$, adjusted for computational viability (as higher values of $\beta$ produce less noisy graphs, so less iterations are needed as well.)
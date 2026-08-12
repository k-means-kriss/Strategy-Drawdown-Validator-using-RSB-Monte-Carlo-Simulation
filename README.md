# Strategy Drawdown Validator using RSB & Monte Carlo Simulation

A quantitative **Strategy Drawdown Validator** built around the **RSB drawdown framework** and Monte Carlo simulation.

The project evaluates whether a trading strategy's observed downside behavior is **statistically normal, deteriorating, or potentially failing** by comparing its realized risk metrics against simulated distributions generated under multiple return models.

---

## 🎯 Core Question

> **Is the strategy experiencing normal statistical pain — or is its risk behavior evidence that the strategy is failing?**

Rather than judging a strategy only by returns or Sharpe ratio, this project evaluates the **shape, magnitude, and persistence of its downside behavior**.

The validator compares the real strategy against:

* **RSB Monte Carlo**
* **Gaussian Monte Carlo**
* **Realistic Monte Carlo**

and converts the resulting percentile analysis into a simple strategy-health classification.

---

# 🧠 Research Foundation

This project is inspired by two research papers.

### 1. RSB Framework

**You are in a drawdown. When should you start worrying?**
*Adam Rej, Philip Seager & Jean-Philippe Bouchaud*

The RSB framework models strategy P&L as a **drifted Brownian motion** and derives the statistical behavior of drawdown depth and drawdown length as a function of the assumed Sharpe ratio.

The central idea is:

> A drawdown should not be judged in isolation; it should be judged against what is statistically plausible for a strategy with the assumed Sharpe ratio.

---

### 2. Monte Carlo Extension

**Drawdown Risk Beyond Brownian Motion: A Monte-Carlo Framework, Non-Gaussian Extensions, and Long Memory**
*Francesco Landolfi*

This research reframes the RSB framework as a transparent Monte Carlo experiment and expands the analysis to four decision-relevant measures:

* Maximum Drawdown
* Maximum Loss
* Final Negative Time
* Longest Recovery Time

It also examines departures from the Gaussian/Brownian assumption.

---


# 🧮 RSB Model

The RSB framework models strategy P&L as:

$$
dPnL = \mu,dt + \sigma,dW
$$

where:

* $\mu$ represents the drift
* $\sigma$ represents volatility
* $W$ is Brownian motion

After normalizing volatility, the Sharpe ratio becomes the key parameter governing expected drawdown behavior.

The original framework focuses on:

* **Drawdown depth**
* **Drawdown length**

and provides statistical thresholds for determining when a drawdown becomes unusually extreme.

---

# 🎲 Monte Carlo Framework

## 1. RSB Monte Carlo

The RSB simulation follows the Brownian-motion framework used in the RSB research.

The goal is to create a theoretical benchmark for how a strategy with a given Sharpe ratio can behave over a specified horizon.

The project then compares the actual strategy against this simulated benchmark.

---

## 2. Gaussian Monte Carlo

The Gaussian simulation assumes returns are generated from a normal distribution.

This provides a conventional benchmark under approximately Gaussian return behavior.

```
Sharpe + Volatility
        ↓
Gaussian Returns
        ↓
Simulated P&L Paths
        ↓
Risk Distributions

```

---

## 3. Realistic Monte Carlo

The project also uses a **realistic empirical Monte Carlo approach**, sampling from the observed return distribution rather than relying only on a Gaussian assumption.

This allows the simulation to retain more of the empirical structure of the strategy's observed returns.

> The current implementation is a historical/empirical simulation rather than a full reproduction of the second paper's AR(1)-GARCH(1,1) + NIG model. The latter is an extension planned for future versions.

---

# 📊 Risk Metrics

The project calculates five key risk measures for every simulated path.

The extended research framework motivates several of these measures as decision-relevant quantities for live strategy monitoring.

---

## 1. Maximum Drawdown

**Function: ****`drawdown(l)`**

Calculates the maximum peak-to-trough decline of each simulated equity path.

Conceptually:

$$
DD_{max} = \max_t(M_t - X_t)
$$

where $M_t$ is the running maximum and $X_t$ is the current P&L.

The function is implemented directly in the notebook as `drawdown(l)`.

### Why it matters

This is the primary measure of **how severe the strategy's worst decline is**.

---

## 2. Maximum Loss

**Function: ****`max_loss(gauss)`**

Measures the deepest cumulative loss relative to the starting point.

Unlike maximum drawdown, which measures losses from a previous peak, maximum loss measures the worst excursion **below the starting level**.

This distinction is consistent with the extended framework's definition of maximum loss.

The notebook implements this through `max_loss(gauss)`.

---

## 3. Days Lower

**Function: ****`days_of_lower(l)`**

Counts how many observations in a path remain below the running high.

This measures how much time the strategy spends **underwater**.

The function is implemented as `days_of_lower(l)`.

### Why it matters

A strategy may have a large drawdown but recover quickly, or a smaller drawdown that lasts for a very long time.

Days Lower helps distinguish these two situations.

---

## 4. Recovery

**Function: ****`recovery_days(l)`**

Measures the maximum number of consecutive observations required to recover to a previous high.

The function records the time spent below each running maximum and returns the longest such period.

### Why it matters

Recovery time measures how much **patience and capital commitment** the strategy requires after a setback.

The research extension similarly defines longest recovery as the longest interval between successive record highs.

---

## 5. Final Negative

**Function: ****`last_neg(gauss)`**

Tracks the final point at which cumulative P&L remains negative.

This is effectively a measure of:

> **How long can the strategy remain below its starting point before finally becoming profitable?**

The notebook implements this through `last_neg(gauss)`.

This is closely related to the extended framework's **final negative time** measure.

---

# ⚙️ Functions Implemented

The core custom functions developed in the notebook are:

| FunctionPurpose    |                                                                |
| ------------------ | -------------------------------------------------------------- |
| `drawdown(l)`      | Calculates maximum drawdown for simulated paths                |
| `max_loss(gauss)`  | Calculates maximum cumulative loss                             |
| `days_of_lower(l)` | Calculates time spent below the running high                   |
| `recovery_days(l)` | Calculates maximum recovery duration                           |
| `last_neg(gauss)`  | Calculates final time at which cumulative P&L remains negative |

These functions form the core **risk-measure extraction layer** of the project.

---

# 🔬 RSB vs Gaussian vs Realistic

The project compares multiple probability models rather than trusting one simulation assumption.

| ModelPurpose     |                                    |
| ---------------- | ---------------------------------- |
| **RSB MC**       | Theoretical Brownian/RSB benchmark |
| **Gaussian MC**  | Normal-return benchmark            |
| **Realistic MC** | Empirical-return benchmark         |

### Why this matters

If a strategy appears extreme under **only one model**, the result may be model-dependent.

If the strategy is extreme under **all three models**, the evidence becomes much stronger that the observed downside behavior is unusual relative to the chosen benchmarks.

---

# 🔍 Why Multiple Metrics?

A strategy can fail in different ways.

### Depth Problem

The strategy may experience:

```
Large Drawdown
Large Maximum Loss

```

but recover relatively quickly.

### Duration Problem

The strategy may have:

```
Moderate Drawdown
Long Time Underwater
Long Recovery

```

The research extension emphasizes exactly this **depth–duration asymmetry**: different strategy structures can distort different risk measures rather than all measures moving together.

Therefore, looking at only maximum drawdown is not enough.

---

# 📈 Percentile Analysis

After calculating each metric across the Monte Carlo simulations, the project constructs empirical distributions.

For every metric, the validator calculates:

* **50th percentile**
* **90th percentile**
* **95th percentile**

These represent progressively more extreme simulated outcomes.

The underlying Monte Carlo methodology is based on applying each risk measure to a large ensemble of equity paths and summarizing the resulting distributions through quantiles.

---

# 🚦 Strategy Health Classification

The core decision system uses the percentile position of the **real strategy's observed metric** relative to the Monte Carlo distribution.

| Actual PercentileClassificationMeaning |                   |                                     |
| -------------------------------------- | ----------------- | ----------------------------------- |
| **< 50th**                             | 🟢 **Healthy**    | Better than most simulated outcomes |
| **50th–90th**                          | 🟡 **Normal**     | Within the expected range           |
| **90th–95th**                          | 🟠 **Borderline** | Risk is deteriorating               |
| **> 95th**                             | 🔴 **Breaking**   | Risk is abnormally high             |

### Interpretation

```
< 50%
   ↓
🟢 HEALTHY
   ↓
50–90%
   ↓
🟡 NORMAL
   ↓
90–95%
   ↓
🟠 BORDERLINE
   ↓
> 95%
   ↓
🔴 BREAKING

```

---

# 🚨 Overall Strategy Rule

The validator is designed around a **risk-first decision rule**:

> **If a critical downside metric exceeds the 95th percentile, the strategy is classified as BREAKING.**

This prevents a strategy from appearing healthy simply because some secondary metrics look good while a critical risk metric is severely abnormal.

For example:

```
Drawdown        → 99.8th percentile 🔴
Max Loss        → 95.3rd percentile 🔴
Days Lower      → 11th percentile   🟢
Recovery        → 7th percentile    🟢
Final Negative  → 100th percentile  🔴

OVERALL STATUS  → 🔴 BREAKING

```

---

# 🧪 Validation Workflow

```
1. Load Strategy Returns
        ↓
2. Build P&L Series
        ↓
3. Generate RSB MC Paths
        ↓
4. Generate Gaussian MC Paths
        ↓
5. Generate Realistic MC Paths
        ↓
6. Calculate Risk Metrics
        ↓
   ├── Drawdown
   ├── Max Loss
   ├── Days Lower
   ├── Recovery
   └── Final Negative
        ↓
7. Calculate 50th / 90th / 95th Quantiles
        ↓
8. Locate Actual Strategy Percentile
        ↓
9. Apply Health Classification
        ↓
10. Final Strategy Decision

```

---

# 📊 Model Comparison

For each horizon, the validator produces comparison tables containing:

```
                RSB MC       Gaussian MC       Realistic MC
              50  90  95     50  90  95        50  90  95
Drawdown
Max Loss
Days Lower
Recovery
Final Negative

```

It also calculates the actual strategy's percentile within each simulation distribution.

This allows the validator to answer:

> **Is the strategy unusual under one model, or consistently unusual across multiple models?**

---

# 🕐 Multiple Time Horizons

The project evaluates the strategy over different horizons, including:

* **1-year validation**
* **10-year validation**

The longer-horizon analysis is particularly useful for observing whether extreme downside characteristics remain unusual over an extended period.

---

# 📌 Example Interpretation

Suppose the actual strategy produces:

| MetricPercentile |        |
| ---------------- | ------ |
| Drawdown         | 99.8th |
| Max Loss         | 95.3rd |
| Days Lower       | 11th   |
| Recovery         | 7th    |
| Final Negative   | 100th  |

The conclusion is **not** that every aspect of the strategy is bad.

Instead:

* Drawdown → 🔴 Breaking
* Max Loss → 🔴 Breaking
* Days Lower → 🟢 Healthy
* Recovery → 🟢 Healthy
* Final Negative → 🔴 Breaking

The strategy therefore has a **depth/downside problem**, rather than primarily a recovery-duration problem.

### Final Classification

# 🔴 BREAKING

---

# 💡 What This Project Actually Tells You

The project is not simply a Monte Carlo price simulator.

It is a **strategy-risk diagnostic system**.

Its purpose is to determine:

> **Whether the strategy's observed downside behavior falls inside the range of statistically plausible outcomes — or whether it has moved into an abnormal risk regime.**

This is closely aligned with the original RSB motivation: a deep or prolonged drawdown can be evidence of bad luck, an overestimated Sharpe ratio, or an inadequate return model, and should therefore act as a precautionary signal rather than being interpreted in isolation.

---

# 🛠️ Technologies

* Python
* NumPy
* Pandas
* SciPy
* Matplotlib
* yFinance
* Monte Carlo Simulation
* Quantitative Risk Management
* Statistical Analysis

---

# 📂 Project Structure

```
STRATEGY-DRAWDOWN-VALIDATOR/
│
├── STRATEGY_DRAWDOWN_VALIDATOR_USING_RSB_AND_MONTE_CARLO_SIMULATION.ipynb
│
├── README.md
│
└── data/
    └── strategy_returns.csv

```

---


---

# 🏗️ Project Architecture

```
                 Strategy Returns
                        │
                        ▼
               Return / P&L Series
                        │
                        ▼
              ┌──────────────────┐
              │  Monte Carlo MCs  │
              └──────────────────┘
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          RSB MC    Gaussian MC  Realistic MC
             │          │          │
             └──────────┼──────────┘
                        ▼
                 Risk Metrics
                        │
                        ▼
                Percentile Analysis
                        │
                        ▼
              Strategy Classification
                        │
         ┌──────────────┼───────────────┐
         ▼              ▼               ▼
      Healthy         Normal       Borderline
                                      │
                                      ▼
                                   Breaking

```
---

# ⭐ Core Takeaway

> ### **The Strategy Drawdown Validator answers one critical question:**
>
> **Is the strategy suffering normal statistical pain — or is its downside behavior abnormal enough to indicate that the strategy may be failing?**

It combines **RSB theory + Monte Carlo simulation + risk metrics + percentile analysis** to turn that question into an objective quantitative validation process.

---

## 📚 Research Papers

* [**You are in a drawdown. When should you start worrying?**](https://arxiv.org/abs/1707.01457) — Adam Rej, Philip Seager & Jean-Philippe Bouchaud
* [**Drawdown Risk Beyond Brownian Motion: A Monte-Carlo Framework, Non-Gaussian Extensions, and Long Memory**](https://arxiv.org/abs/2608.00127) — Francesco Landolfi

# Dynamic Sparse Predictive Agent (DSP-A) with Residual-Triggered Topology Adaptation and Counterfactual Pruning (v21.1 Revised)

[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg)](LICENSE)
[![Engine: Substrate--Independent](https://img.shields.io/badge/Engine-Substrate--Independent-FF007F.svg)]()

DSP-A is an online, adaptive learning system that reconfigures its own dataflow topology (which units and connections are active) based on streaming prediction error. It functions as a sparse online neural network with two extra mechanisms layered on top of standard gradient learning:

1. A **growth/rewiring rule** that activates or relocates hidden units when prediction error stays high.
2. A **pruning rule** that removes low-value units when the network hits a memory budget, using counterfactual replay to estimate each unit's contribution.

---

## 🏛️ GLOBAL ARCHITECTURE DATAFLOW

```text
[Global Environment State x_t ∈ X] ──(Lossy Random Projection φ)──> [Latent Manifold State z_t ∈ Z]
                                                                                │
                                                                   (Forward Predictor f_A)
                                                                                │
                                                                                ▼
[Sandboxed Mutation Audit Tier Φ_U] <──(SPE > dynamic_threshold)── [Empirical Residual Covariance Σ̂_t]
                │
     (Structural Loss check:
  L(F_candidate) < L(F_active))
                │
                ▼
[Structural Evolution F_t ∪ {A*}]
                │
    (Rigid Memory Ceiling M_max)
                │
                ▼
[Counterfactual Ablation Pruning] ───────────────────────────────> [Adaptive Cooldown Threshold Inflation]
```

---

## 1. PURPOSE & ARCHITECTURAL SCOPE

The framework operates entirely within the boundaries of continuous latent manifolds regulated by discrete topological boolean masking tensors. DSP-A is a bounded-memory, bounded-compute online learner designed to operate in non-ergodic environments.

* **Environment ($\mathcal{E}$):** A non-ergodic dynamical system on a latent state space $\mathcal{X} \subseteq \mathbb{R}^D$, not directly observed.
* **Observation manifold ($\mathcal{Z}$):** The agent observes $\mathcal{E}$ only through a lossy projection $\phi: \mathcal{X} \rightarrow \mathcal{Z}$, $\mathcal{Z} \subseteq \mathbb{R}^d$ ($d \ll D$):
  $$z_t = \phi(x_t) \in \mathbb{R}^d$$
* **Resource tuple:** Computation is constrained by $\langle M_{\max}, \tau, \mathcal{F}_t \rangle$ — a hard memory ceiling $M_{\max}$ (bits needed to serialize the active graph), a per-step compute time budget $\tau$, and the currently active DAG configuration $\mathcal{F}_t$.

---

## 2. ONLINE RESIDUAL COVARIANCE

Let $f_{\mathcal{A}}: \mathcal{Z} \times \mathcal{A} \rightarrow \mathcal{Z}$ be the network's forward prediction function under active configuration $\mathcal{F}_t$. Define the rolling residual covariance over a window $K$:

$$\hat{\Sigma}_t := \frac{1}{K}\sum_{i=0}^{K-1} \left( z_{t-i} - f_{\mathcal{A}}(z_{t-i-1}, a_{t-i-1}) \right) \left( z_{t-i} - f_{\mathcal{A}}(z_{t-i-1}, a_{t-i-1}) \right)^T$$

**Properties:**
1. $\hat{\Sigma}_t = \hat{\Sigma}_t^T$, $\hat{\Sigma}_t \succeq 0$.
2. $\text{Tr}(\hat{\Sigma}_t)$ is the **Squared Prediction Error (SPE)** — a plain measure of how much of the environment's behavior the current topology fails to capture.
3. At $t=1$, initialize the trace to $\frac{1}{d}\|z_1 - f_{\mathcal{A}}(\cdot)\|_2^2$ to avoid a divide-by-small-sample blowup on the first step.

---

## 3. STRUCTURAL LOSS

$$\mathcal{L}(\mathcal{F}_t) = \alpha \cdot \text{Tr}(\hat{\Sigma}_t) + \beta \cdot \mathcal{L}_{\text{MDL}}(\mathcal{F}_t)$$

$\alpha, \beta \in \mathbb{R}^+$ are fixed regularization weights; $\mathcal{L}_{\text{MDL}}(\mathcal{F}_t)$ is the serialized bit-length of the active graph. Under an isotropic Gaussian noise assumption with variance $\sigma^2$, the per-step surprise signal used to trigger mutations,
$$\Delta\kappa_E = \frac{1}{2\sigma^2}\|z_t - f_{\mathcal{A}}(z_{t-1}, a_{t-1})\|_2^2,$$
is exactly the negative log-likelihood of the observation under that assumption. This is a labeled equivalence, not a claim that the agent is doing Bayesian inference — it's a squared-error trigger with a probabilistic reading available if useful.

---

## 4. PARAMETER UPDATES ON THE ACTIVE SUB-NETWORK

Weight updates apply only to parameters whose connectivity mask is currently $1$.

### 4.1. Active Jacobian
For $\theta_{\text{active}} \in \mathbb{R}^p$ (flattened weights at active mask coordinates):
$$J = \nabla_{\theta_{\text{active}}} f_{\mathcal{A}}(z_{t-1}, a_{t-1}) \in \mathbb{R}^{d \times p}$$
computed via finite differences, costing $O(p_{\text{active}})$ per step — this bounds how large the active sub-network can be within the time budget $\tau$.

### 4.2. Damped Gauss-Newton Update
$$F := \frac{1}{\sigma^2} J^T J + \lambda I_p, \qquad \lambda = 10^{-4}$$
$$\theta_{\text{active}, t+1} = \theta_{\text{active}, t} - \eta_{\text{adapted}} \, F^{-1} g_{\text{Euclidean}}$$
$\lambda I_p$ guarantees $F$ is invertible; $g_{\text{Euclidean}}$ is the raw empirical-risk gradient; $\eta_{\text{adapted}}$ is a damped, adaptively scaled learning rate.

---

## 5. TOPOLOGY CONTROL LOOP

Topology changes are read directly off the connectivity masks ($\mathcal{M}_1, \mathcal{M}_2$) rather than tracked with a separate counter, to avoid ledger/counter drift bugs.

### 5.1. Mask Queries
$$\mathcal{V}_{\text{active}} = \{ i \in \mathcal{V}_{\text{capacity}} \mid \sum_{j} \mathcal{M}_{1}(j, i) > 0 \}$$
$$\mathcal{V}_{\text{inactive}} = \{ i \in \mathcal{V}_{\text{capacity}} \mid \sum_{j} \mathcal{M}_{1}(j, i) = 0 \}$$

### 5.2. Sandboxed Mutation Check
To prevent the network from "gaming" the loss by deleting the parts of itself that measure error, candidate mutations are evaluated in an isolated copy before being committed:

```
[ Active Network ] ── SPE > threshold ──> [ Candidate Mutation (sandboxed) ]
        ▲                                              │
        │                               Try: activate unit, or rewire
        │                                              │
        └────────── commit only if ────────────────────┘
                  L(F_candidate) < L(F_active)
```

* **Activation:** if SPE breaches the activation threshold, flip the mask of the first available index in $\mathcal{V}_{\text{inactive}}$ to $1$.
* **Rewiring:** if growth stagnates, hibernate the oldest node in $\mathcal{V}_{\text{active}}$ (mask $\rightarrow 0$) and activate a fresh index from $\mathcal{V}_{\text{inactive}}$ instead.

A mutation is only committed if it actually reduces $\mathcal{L}(\mathcal{F})$ on the sandboxed trial — this is the mechanism that prevents degenerate self-deleting shortcuts.

---

## 6. COUNTERFACTUAL ABLATION PRUNING

When $\mathcal{L}_{\text{MDL}}(\mathcal{F}_t) \ge M_{\max}$, the system must free capacity. Each hidden unit $v_i$ is scored by re-running the stored history with that unit's mask column zeroed out:

$$\omega_i = \frac{\left| \text{Tr}(\hat{\Sigma}_{t \setminus v_i}) - \text{Tr}(\hat{\Sigma}_t) \right|}{\mathcal{L}_{\text{MDL}}(v_i)}$$

$\hat{\Sigma}_{t \setminus v_i}$ is computed by replaying `history_inputs` against `history_targets` with unit $i$ ablated. The unit with the **smallest** $\omega_i$ (least error-reduction per bit of description length) is permanently removed. This prunes based on measured contribution rather than recency.

---

## 7. ADAPTIVE PACING FOR THE MUTATION THRESHOLD

After a pruning pass removes multiple units, the network is momentarily unstable: SPE can spike simply because capacity just dropped, not because the environment changed. Without correction this can trigger a cascade of unnecessary re-activations.

To prevent that, the activation threshold is temporarily raised in proportion to how much structure was just removed, then decays back to baseline:

$$\text{threshold}_{\text{dynamic}} = \text{threshold}_{\text{base}} + \gamma \cdot \Delta\text{bits}_{\text{removed}}$$

where $\Delta\text{bits}_{\text{removed}}$ is the description-length freed by the last pruning pass, and $\gamma \approx 2.85$ is a tunable gain fit empirically rather than derived from a physical constant. This functions as an adaptive cooldown / momentum term on the mutation trigger.

---

## 8. VERIFIABLE SYNTHETIC BENCHMARK PROTOCOL

The agent is tested against a synthetic signal generator that alternates dynamics:
* **Regime A:** $z_{\text{target}} = \tanh(x W_A) + \epsilon$
* **Regime B:** $z_{\text{target}} = \sin(2x W_B) + \epsilon$
where $\epsilon \sim \mathcal{N}(0, 0.05^2)$.

### Baselines (Fixed budget $M_{\max} = 2000$ bits)
1. **Fixed-Topology MLP:** Masks locked at full capacity; standard full-parameter updates.
2. **Random-Restart MLP:** Static topology, full parameter re-initialization on threshold breach.
3. **LRU Pruning Agent:** Same growth rule, but LRU deletion instead of counterfactual ablation.

### Metrics
* **Adaptation Error ($\mathcal{E}_{\text{adapt}}$):** mean SPE over the 50 steps after a regime switch.
* **Memory Efficiency ($\eta_M$):** description length per unit of inverse prediction variance.
* **Structural Churn ($\Omega_S$):** topology changes per 100 steps.

---

## 📁 Repository Structure

```text
DSP-A/
│
├── LICENSE                  # AGPL-3.0 License
├── README.md                # System specification architecture document (v21.1 Revised)
│
├── web_portal/              # Frontend presentation layer
│   └── index.html           # Single-file HTML5/JS dashboard with live DSP-A simulator
│
└── core_engine/             # Backend computational engine
    ├── __init__.py          # Module exports
    └── stream_physics.py    # Python implementation of the DSP-A engine & benchmark suite
```

---

## 🚀 Getting Started

### 1. Launching the Frontend Visualizer
The web portal simulates the DSP-A operations entirely client-side:
1. Navigate to the `web_portal/` directory.
2. Open `index.html` in any modern web browser.
3. Configure your API credentials inside the credentials panel to authorize the Y3 OS Console.

### 2. Running the Backend Engine
The Python simulator evaluates the matrices, natural gradients, and runs the standardized benchmark comparison suite:
1. Install `numpy` dependency:
   ```bash
   pip3 install numpy
   ```
2. Run the benchmark suite:
   ```bash
   python3 core_engine/stream_physics.py
   ```

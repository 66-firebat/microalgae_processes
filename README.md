| Step | What Happens | Why It Matters |
|---|---|---|
| **1. Collect data** | Gather input-output time-series from the real reactor or a validated simulation — states (concentration, temperature) and inputs (feed rate, coolant flow), sampled over time. | The model can only be as good as the data. Needs enough variation (excitation) in the inputs for the system to reveal its true nonlinear behavior — steady-state data alone won't work. |
| **2. Identify the model** | Build the digital twin's dynamic core using either **SINDy**[^2] or the **Koopman operator**[^3] — see comparison table below. | Produces the model that everything downstream (control, updating) depends on. |
| **3. Add control (MPC)** | Wrap the identified model inside a Model Predictive Controller — Lyapunov-based/nonlinear MPC for a SINDy model, or linear MPC for a Koopman model — so the twin doesn't just describe the reactor, it actively drives real-time decisions. | Turns the twin from a passive simulator into something that can actually make setpoint/control decisions, with formal stability guarantees achievable in either approach (more straightforwardly for the linear Koopman case). |
| **4. Update online** | Continuously refresh the model as conditions drift — either via an Extended Kalman Filter (treating model coefficients as extra states to track) or periodic re-fitting. | Keeps the twin synchronized with the real plant over time (catalyst decay, fouling, biological drift), instead of staying frozen at its original identification and slowly going stale. |

---

## Step 2: Two methods, SINDy vs. Koopman Operator

| Dimension | SINDy[^2] | Koopman Operator[^3] |
|---|---|---|
| **Core Utility** | The true governing equation is a *sparse combination of nonlinear terms* in the original state variables. | The true dynamics become *linear* if you track the right functions ("observables") of the state instead of the state itself. |
| **Fitting Objective** | A sparse coefficient matrix `Ξ` such that `ẋ = Θ(x,u)·Ξ` — a nonlinear ODE in the original state `x`. | A linear operator `K` (or `A`, `B` with control) such that `g(x_{t+1}) ≈ K·g(x_t)` — linear dynamics in a *lifted* space of observables `g(x)`. |
| **Fitting method** | Sequential thresholded least squares (sparse regression) over a candidate term library. | Extended Dynamic Mode Decomposition (EDMD) — least-squares fit of a linear map between snapshot pairs of the observables. |
| **Core Disparities** | A library of candidate nonlinear terms (polynomials, exponentials, domain-specific kinetics like Arrhenius or Monod/Droop). | A dictionary of observable functions `g(x)` to lift the state into. |
| **Resulting model form** | A short, explicit nonlinear ODE — human-readable. | A (potentially large) linear matrix acting on lifted coordinates — not directly readable as physics. |
| **Interpretability** | High — surviving terms map onto recognizable physical/biological mechanisms. | Low to moderate — physical meaning has to be extracted from the operator's spectral structure (eigenfunctions), if at all. |
| **Robustness to a bad library/dictionary choice** | Self-correcting: an irrelevant candidate term just gets thresholded to zero. | Fragile: a poorly chosen observable dictionary degrades the linear approximation silently, without a clear failure signal. |
| **Downstream control problem** | Nonlinear MPC — nonconvex optimization at every timestep; needs extra machinery (e.g. Lyapunov constraints) for stability guarantees. | Linear MPC — convex quadratic program; fast, reliable, mature solvers, easier formal stability guarantees. |
| **Extrapolation behavior** | Generally better *if* the library contains the right physical terms — the model stays grounded in known kinetics. | Often only locally valid; a single global linear approximation can struggle outside the operating envelope it was fit on (e.g. multiple steady states, thermal runaway, photoinhibition). |
| **Optimal Use Cases** | Systems where you have some idea of the underlying physics/kinetics and want a twin you can inspect and trust. | Systems where fast, provably stable real-time control matters more than reading the model, and a reasonable observable dictionary is available. |
| **Cross-Compatibility** | Yes — SINDy-style sparse regression can be used to *discover* good Koopman observables, blending SINDy's interpretability with Koopman's easy-to-control linear structure. | Same combination, viewed from the other side. |

---

## What is SINDy?

**SINDy** = **S**parse **I**dentification of **N**onlinear **Dy**namics — originally introduced by Brunton, Proctor, and Kutz (2016)[^2], a method for discovering the governing differential equations of a system directly from data, rather than deriving them by hand or fitting an opaque model like a neural network.

### The core idea
Most physical systems are governed by an equation of the form `ẋ = f(x, u)`, where `x` is the state and `u` is the input. SINDy assumes the true `f` is probably a *short combination of a few simple, recognizable terms* — not something requiring millions of parameters to approximate.

### SINDy Framework
1. Measure the state trajectory `x(t)` and inputs `u(t)`, and estimate the derivative `ẋ(t)`.
2. Build a "library" of candidate terms that might plausibly appear in the true equation — constants, `x`, `u`, products like `x²`, `xu`, and optionally domain-specific terms (e.g. an Arrhenius rate term for a chemical reactor, or Monod/Droop growth terms for a bioreactor).
3. Run a sparse regression that fits `ẋ` against that library while forcing most coefficients to exactly zero — fit, threshold out small coefficients, refit, repeat until stable.
4. Whatever terms survive with nonzero coefficients *are* the discovered model.

### The Purpose of Sparsity
A dense fit (keeping every library term) would match training data but wouldn't generalize and wouldn't mean anything — just a curve fit dressed up as a model. Forcing sparsity turns the output into something closer to an actual physical law: a handful of interpretable terms doing real work, rather than a black box.

### Applications to a Digital Twin
- Data-driven (no need to already know the mechanism) but interpretable (unlike a neural net, you can read what it found).
- Output is a small set of coefficients, not millions of weights — cheap to re-fit or update online, which is exactly what a self-updating twin needs.
- Main limitation: it can only discover what's in your candidate library. If you don't include an Arrhenius or Monod/Droop term as a candidate, SINDy can't find that kinetic form — it'll approximate it clumsily with whatever generic terms you did supply. This is why seeding the library with known kinetics (rather than raw polynomials alone) matters so much for chemical or biological reactors.

---

## What is the Koopman operator?

**Koopman operator theory**[^3] takes the opposite strategy from SINDy: instead of finding a nonlinear equation directly in the original state variables, it looks for a *linear* operator `K` that exactly evolves any chosen function `g(x)` of the state: `g(x_{t+1}) = K·g(x_t)`. The nonlinearity doesn't disappear — it gets absorbed into the choice of `g`, so that whatever you're tracking evolves linearly.

### Koopman Framework
1. Choose a dictionary of observable functions `g(x)` — e.g. the states themselves plus various nonlinear functions of them.
2. Fit the best linear operator (via EDMD) that advances those observables forward in time: `g(x_{t+1}) ≈ A·g(x_t) + B·u_t` when control inputs are included.
3. That linear operator, acting in the lifted observable space, becomes the model's dynamic core.

### Purpose
Linear dynamics make the downstream control problem dramatically easier — convex optimization, mature solvers, and much simpler stability guarantees than the Lyapunov-constrained nonlinear MPC a SINDy model requires.

### Applications to a Digital Twin
- Turns the hardest downstream problem — real-time control — into a convex optimization, which is dramatically cheaper and more reliable to solve at every control step than the nonconvex problem a nonlinear SINDy model creates.
- Comes with a much more mature toolbox for formal guarantees: standard linear systems theory (LQR, linear Kalman filters, robust/H∞ control) all becomes directly applicable once the model is linear, rather than needing the extra Lyapunov-constraint machinery a nonlinear model requires just to prove stability.
- Scales more predictably to multi-input, multi-output reactors — a linear state-space model composes and couples with other linear subsystems (e.g. a downstream separation unit) far more easily than stitching together several nonlinear ODEs.
- Still adapts online the same way a SINDy twin does — the fitted operator `K` (or `A`, `B`) can be re-estimated via EKF or periodic re-fitting exactly like SINDy's coefficient matrix `Ξ`, so it doesn't lose the "living, updating twin" property.
- Main limitation: the whole approach lives or dies on the observable dictionary. Unlike SINDy, there's no built-in sparsity step to protect you from a bad choice — if your dictionary doesn't actually span a Koopman-invariant subspace for the true system, the linear approximation degrades quietly rather than failing in an obvious, diagnosable way. This is a real risk for reactors with strongly nonlinear behavior (multiple steady states, thermal runaway, photoinhibition) unless the dictionary is chosen carefully around the known physics.


---

[^1]: Bhadriraju, B., Narasingam, A., & Kwon, J. S.-I. (2019). Machine learning-based adaptive model identification of systems: Application to a chemical process. *Chemical Engineering Research and Design*, 152, 372–383. https://doi.org/10.1016/j.cherd.2019.10.004 (Open-access copy: https://oaktrust.library.tamu.edu/server/api/core/bitstreams/d42d7090-1a06-49df-abd4-de18f97a8cc1/content)

[^2]: Brunton, S. L., Proctor, J. L., & Kutz, J. N. (2016). Discovering governing equations from data by sparse identification of nonlinear dynamical systems. *Proceedings of the National Academy of Sciences*, 113(15), 3932–3937. https://doi.org/10.1073/pnas.1517384113 (Open-access copy: https://pmc.ncbi.nlm.nih.gov/articles/PMC4839439)

[^3]: Brunton, S. L., Brunton, B. W., Proctor, J. L., & Kutz, J. N. (2016). Koopman invariant subspaces and finite linear representations of nonlinear dynamical systems for control. *PLOS ONE*, 11(2), e0150171. https://doi.org/10.1371/journal.pone.0150171 (Open access: https://pmc.ncbi.nlm.nih.gov/articles/PMC4769143)

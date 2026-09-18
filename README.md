| Step | What Happens | Why It Matters |
|---|---|---|
| **1. Collect data** | Gather input-output time-series from the real reactor or a validated simulation — states (concentration, temperature) and inputs (feed rate, coolant flow), sampled over time. | The model can only be as good as the data. Needs enough variation (excitation) in the inputs for the system to reveal its true nonlinear behavior — steady-state data alone won't work. |
| **2. Identify the model (SINDy)** | Apply SINDy to discover a sparse, interpretable ODE model from a library of candidate nonlinear terms (polynomials, exponentials, kinetic terms). This becomes the digital twin's dynamic core.[^1] | Produces a short, human-readable equation instead of a black-box model — you can see exactly which physical/biological terms the model relies on, and sanity-check it against known kinetics. |
| **3. Add control (MPC)** | Wrap the identified model inside a Model Predictive Controller — Lyapunov-based MPC in the OASIS line, or a standard MPC — so the twin doesn't just describe the reactor, it actively drives real-time decisions. | Turns the twin from a passive simulator into something that can actually make setpoint/control decisions, with (in the Lyapunov case) a formal stability guarantee despite using a learned rather than first-principles model. |
| **4. Update online** | Continuously refresh the model as conditions drift — either via an Extended Kalman Filter (treating model coefficients as extra states to track) or periodic re-fitting. | Keeps the twin synchronized with the real plant over time (catalyst decay, fouling, biological drift), instead of staying frozen at its original identification and slowly going stale. |

---

## What is SINDy?

**SINDy** = **S**parse **I**dentification of **N**onlinear **Dy**namics (Brunton, Proctor & Kutz, 2016) — a method for discovering the governing differential equations of a system directly from data, rather than deriving them by hand or fitting an opaque model like a neural network.

### The core idea
Most physical systems are governed by an equation of the form `ẋ = f(x, u)`, where `x` is the state and `u` is the input. SINDy assumes the true `f` is probably a *short combination of a few simple, recognizable terms* — not something requiring millions of parameters to approximate.

### How it works
1. Measure the state trajectory `x(t)` and inputs `u(t)`, and estimate the derivative `ẋ(t)`.
2. Build a "library" of candidate terms that might plausibly appear in the true equation — constants, `x`, `u`, products like `x²`, `xu`, and optionally domain-specific terms (e.g. an Arrhenius rate term for a chemical reactor, or Monod/Droop growth terms for a bioreactor).
3. Run a sparse regression that fits `ẋ` against that library while forcing most coefficients to exactly zero — fit, threshold out small coefficients, refit, repeat until stable.
4. Whatever terms survive with nonzero coefficients *are* the discovered model.

### Why sparsity is the point
A dense fit (keeping every library term) would match training data but wouldn't generalize and wouldn't mean anything — just a curve fit dressed up as a model. Forcing sparsity turns the output into something closer to an actual physical law: a handful of interpretable terms doing real work, rather than a black box.

### Why it's the right fit for a digital twin
- Data-driven (no need to already know the mechanism) but interpretable (unlike a neural net, you can read what it found).
- Output is a small set of coefficients, not millions of weights — cheap to re-fit or update online, which is exactly what a self-updating twin needs.
- Main limitation: it can only discover what's in your candidate library. If you don't include an Arrhenius or Monod/Droop term as a candidate, SINDy can't find that kinetic form — it'll approximate it clumsily with whatever generic terms you did supply. This is why seeding the library with known kinetics (rather than raw polynomials alone) matters so much for chemical or biological reactors.

---

[^1]: Bhadriraju, B., Narasingam, A., & Kwon, J. S.-I. (2019). Machine learning-based adaptive model identification of systems: Application to a chemical process. *Chemical Engineering Research and Design*, 152, 372–383. https://doi.org/10.1016/j.cherd.2019.10.004 (Open-access copy: https://oaktrust.library.tamu.edu/server/api/core/bitstreams/d42d7090-1a06-49df-abd4-de18f97a8cc1/content)

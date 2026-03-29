# Simulating Open Quantum Systems Using Hamiltonian Simulations

This is my project for the **QIntern 2024** program under **QWorld**. The goal was to study and implement a method to simulate Lindblad (open quantum system) dynamics using Hamiltonian simulations, based on the paper by Ding, Li, and Lin (arXiv:2311.15533).

---

## Background

When we talk about quantum systems in the real world, they are almost never perfectly isolated. They interact with their environment, which leads to dissipation, noise, and decoherence. The standard way to describe this is through the **Lindblad master equation**:

$$\frac{d\rho}{dt} = -i[H, \rho] + \sum_j \left( V_j \rho V_j^\dagger - \frac{1}{2}\{V_j^\dagger V_j, \rho\} \right)$$

The problem is, simulating this on a quantum computer is not straightforward. Many existing algorithms require complicated control logic, amplitude amplification, or ancilla-heavy procedures. The paper I studied proposes something cleaner in order to reduce the whole thing to a **Hamiltonian simulation** problem, which is much better understood.

---

## The Three-Step Pipeline

The core idea of the method is a three-step derivation:

```
Lindblad equation  →  SDE unraveling  →  Kraus form  →  Dilated Hamiltonian
```

**Step 1 — Unravel into a Stochastic Schrödinger Equation (SSE)**

The Lindblad equation can be "unraveled" into a stochastic differential equation for a pure state $|\psi_t\rangle$:

$$d|\psi_t\rangle = \left(-iH - \frac{1}{2}\sum_j V_j^\dagger V_j\right)|\psi_t\rangle\, dt + \sum_j V_j |\psi_t\rangle\, dW_t^j$$

where $dW_t^j$ are independent Wiener processes. If you average the resulting density matrices over many trajectories, you recover the Lindblad solution exactly.

**Step 2 — Discretize and write in Kraus form**

By applying classical numerical SDE schemes (like the Euler–Maruyama method) and taking expectations, we get a Kraus representation of one time step:

$$\rho_{n+1} = \sum_j F_j \rho_n F_j^\dagger$$

Higher-order schemes (using Itô–Taylor expansion) give higher-order Kraus forms with more operators.

**Step 3 — Lift to a dilated Hamiltonian**

Instead of implementing the Kraus form directly, the method constructs a **dilated (enlarged) Hamiltonian** $\tilde{H}$ in a bigger Hilbert space (system + ancilla qubits), such that:

$$\rho_{n+1} = \text{Tr}_A\!\left[ e^{-i\sqrt{\Delta t}\,\tilde{H}} \left(|0^{a_k}\rangle\langle 0^{a_k}| \otimes \rho_n\right) e^{+i\sqrt{\Delta t}\,\tilde{H}} \right]$$

This is just a **Hamiltonian simulation** followed by **tracing out the ancilla** — no postselection, no amplitude amplification, success probability = 1 at every step.

The dilated Hamiltonian has the block structure:

$$\tilde{H} = \begin{pmatrix} H_0 & H_1^\dagger & H_2^\dagger & \cdots \\ H_1 & 0 & 0 & \cdots \\ H_2 & 0 & 0 & \cdots \\ \vdots & & & \ddots \end{pmatrix}$$

where each $H_j$ is built from polynomials of the system Hamiltonian $H$ and the jump operators $V_j$.

---

## My Analysis: Amplitude Damping Channel

Before working on the full notebook, I wrote a separate analysis focusing on the **amplitude damping (AD) channel** — one of the simplest and most physically meaningful open quantum system models. It describes spontaneous emission: an excited qubit $|1\rangle$ decays to the ground state $|0\rangle$ with some rate $\gamma$.

In my analysis (`My_Analysis.pdf`), I worked through:

- The Kraus operator representation of the AD channel
- How to derive the Lindblad master equation from the Kraus operators in the infinitesimal limit
- The formal derivation of the Stochastic Schrödinger Equation (SSE) that unravels the Lindblad equation
- Solving for the drift and diffusion operators ($A$ and $B$) using the homodyne-detection ansatz

This gave me a concrete grounding in the physics before implementing the general pipeline.

---

## What's in the Notebook

The notebook (`lindblad_hamiltonian_simulation.ipynb`) implements the full pipeline for two physical models:

### Model 1 — Amplitude Damping (AD) Channel

A single-qubit model for spontaneous emission, with jump operator $V = \sqrt{\gamma}\,\sigma_-$.

- Implements the Kraus operators $K_0$ and $K_1$
- Solves the Lindblad equation using 4th-order Runge-Kutta (used as "exact" reference)
- Builds the first- and second-order Kraus schemes
- Constructs the dilated Hamiltonian and simulates via matrix exponentiation
- Simulates the SSE (stochastic Schrödinger equation) trajectories and averages them
- Verifies the Stinespring dilation error scales as $O(\Delta t^2)$

### Model 2 — Transverse-Field Ising Model (TFIM) with Damping

A multi-qubit spin chain:

$$H = -\sum_{i=1}^{m-1} Z_i Z_{i+1} - Z_m Z_1 - g\sum_{i=1}^m X_i$$

with damping operators $V_j = \sqrt{\gamma}(X_j - iY_j)/2$ for each site.

- Tests the same first- and second-order Kraus schemes on a 4-site system ($m=4$, $g=1$, $\gamma=0.1$)
- Measures convergence rate by varying $\Delta t$ and checking global error in trace distance
- Plots the ground-state overlap over time

### Convergence Results

| Model | Method | Error scaling |
|-------|--------|--------------|
| AD channel | 1st-order Kraus | $O(\Delta t)$ |
| AD channel | 2nd-order Kraus | $O(\Delta t^2)$ |
| AD channel | Dilated Hamiltonian | $O(\Delta t)$ |
| TFIM ($m=4$) | 1st-order Kraus | $O(\Delta t)$ |
| TFIM ($m=4$) | 2nd-order Kraus | $O(\Delta t^2)$ |
| TFIM ($m=4$) | Dilated Hamiltonian | $O(\Delta t)$ |

The numerical results match the theoretical predictions from the paper.

---

## Key Properties of the Method

- **CPTP at every step** — each update is completely positive and trace preserving, so the density matrix stays physical
- **Success probability = 1** — no need for postselection or amplitude amplification
- **Reduces to Hamiltonian simulation** — can directly plug in any Hamiltonian simulation algorithm (Trotterization, quantum signal processing, etc.)
- **Generalizes to time-dependent Lindbladians** — handles driven open quantum systems, unlike many existing methods
- **Systematically improvable** — $k$-th order scheme requires $O(k\log(J+1))$ ancilla qubits

---

## Files

| File | Description |
|------|-------------|
| `lindblad_hamiltonian_simulation.ipynb` | Main notebook implementing the full pipeline |
| `My_Analysis.pdf` | My written analysis of the AD channel — Kraus operators, Lindblad derivation, SSE derivation |

---

## Reference

> Zhiyan Ding, Xiantao Li, Lin Lin.  
> *Simulating Open Quantum Systems Using Hamiltonian Simulations.*  
> arXiv:2311.15533v3, 2024.

---

## About

This project was done as part of **QIntern 2024**, an 3months internship program organized by **QWorld**. I was doing this in 2024 where I was still an undergraduate student that moment, and this was my first time working seriously with open quantum systems and quantum simulation algorithms. It was quite challenging especially in understanding the stochastic calculus parts, but I learned a lot from going through the paper carefully and trying to implement everything from scratch.

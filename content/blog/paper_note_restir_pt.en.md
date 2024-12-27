+++
title = "Paper Note: Generalized Resampled Importance Sampling - ReSTIR PT"
date = "2024-10-17 15:49:21"
authors = ["茨月"]
+++

[Continuing from the previous discussion](https://blog.zcy.moe/blog/paper-note-gris/), ReSTIR PT uses ReSTIR to reuse paths. Since the path spaces of different pixels are not the same, we must find an appropriate way to reuse paths—this corresponds to the shift mapping in GRIS theory.

<!-- more -->

## Formal Definition of Shift Mapping in ReSTIR PT

For pixel \\(k\\) and pixel \\(j\\), \\(T: \Omega_k \to \Omega_j\\) is a mapping from the path space of \\(k\\) to the path space of \\(j\\):

<p>
$$
T\left(\left[ 
    \textbf{x}_0, \textbf{x}_1, \textbf{x}_2, \textbf{x}_3, ... \right]\right)
 = \left[ \textbf{y}_0, \textbf{y}_1, \textbf{y}_2, \textbf{y}_3, ... \right]
$$
</p>

Note that \\(\mathbf{y}_0\\) is the camera, \\(\mathbf{y}_1\\) is the primary ray-traced intersection, and what we need to map is \\(\mathbf{y}_n, n \geq 2\\).

- The shift mapping is a bijection from a subset of \\(\Omega_k\\) to a subset of \\(\Omega_j\\)—if \\(T_{k\to j}(\overline{\mathbf{x}}) = \overline{\mathbf{y}}\\), then \\(T_{j\to k}(\overline{\mathbf{y}}) = \overline{\mathbf{x}}\\).

- At the same time, we want the mapped path to be as similar as possible to the original path, i.e., \\(f_k(T_k(\mathbf{\overline{x}})) \approx f_j(\overline{\mathbf{x}})\\) and \\(\left| \frac{\partial T_k}{\partial \overline{\mathbf{x}}} \right| \approx 1\\).

## Available Shift Mapping Strategies

- **Vertex Reuse (Vertex Copy / Reconnection):**

   Directly reuse the vertices of the original path, \\(y_{i+1} = x_{i+1}\\) (reuse the entire path starting from \\(i+1\\)).

   - Only applicable when \\(x_i, x_{i+1}, y_{i+1}\\) are all on rough surfaces.

- **Half-Vector Reuse (Half-Vector Copy):**

   Convert the half-vector of the base path to the local tangent space, reuse it for the offset path, and then trace \\(y_{i+1}\\).

   - Can handle reflections.

- **Direction Reuse (Direction Copy):**

   Reuse the outgoing direction of the base path for the offset path.

   - Typically used for sampling environment maps.

- **Random Number Reuse (Random Replay):**

   Reuse the random numbers of the base path at the current bounce to sample \\(y_{i+1}\\).

   - Computationally expensive.

   - Often performs similarly to half-vector or direction reuse.

- **Manifold Exploration:**

   See [Lehtinen et al. 2013](https://mediatech.aalto.fi/publications/graphics/GMLT/), [Jakob and Marschner 2012](https://www.cs.cornell.edu/projects/manifolds-sg12/).



+ **ReSTIR GI Strategy:**

   ~~Although back then humans didn’t have GRIS theory, in hindsight we can say~~ it directly records outgoing radiance on probes to sample single-bounce GI, which essentially corresponds to unconditional \\(y_2 = x_2\\) reconnection shift.

+ **ReSTIR PT Strategy:**

   Combines random replay and reconnection [\[Hua et al. 2019\]](https://profs.etsmtl.ca/agruson/publication/2019_gradientstar/). Reuse vertices when reconnection is suitable; otherwise, reuse random numbers.

   The core contribution of the latter half of the paper is proposing an effective heuristic to determine reuse conditions.

## When to Reuse Vertices?

The authors propose two new criteria to decide when vertices cannot be reused.

### Distance Criterion

It is observed that in the rendering equation:

<p>
$$
L_o(\mathbf{x}, \omega_o) = L_e(\mathbf{x}, \omega_o) + \int_{\Omega} f_r(\mathbf{x}, \omega_i, \omega_o) L_i(\mathbf{x}, \omega_i) G(\mathbf{x}, \mathbf{x'}) V(\mathbf{x}, \mathbf{x'}) \text{d}\omega_i
$$
</p>

the geometric term:

<p>
$$
G(\mathbf{x}, \mathbf{x'}) = \frac{ \cos \theta_x \cos \theta_{x'} }{ |\mathbf{x} - \mathbf{x'}|^2 }
$$
</p>

contains an inverse-square dependence on the distance between vertices. When two vertices are very close, this results in a large weight, which can lead to high variance. Therefore, the distance criterion is defined such that reuse is allowed only when:

<p>
$$
\|\mathbf{x}_i - \mathbf{x}_{i+1} \| \geq d_{\max}
$$
</p>

In other words, "if the distance between \\(\mathbf{x}_{i+1}\\) and \\(\mathbf{x}_i\\) is less than the threshold, the vertex cannot be reused."

### Lobe Criterion

When inspecting surface roughness, we encounter BSDFs with multiple lobes (e.g., a diffuse surface with a clear coat). During evaluation, only one lobe is selected at a time. Reconnection is allowed only when the roughness of \\(x_i, x_{i+1}, y_{i+1}\\) is sufficiently high. Since only one lobe is selected at a time, the path must additionally record the lobe ID for each path vertex.

> The hybrid mapping ultimately used performs reconnection shift when the above two conditions are met and random replay when they are not.

## Jacobian Matrix of Shift Mapping

Define the unit vector from \\(\mathbf{x}_i\\) to \\(\mathbf{x}_{i+1}\\) as \\(\omega_i^\mathcal{X}\\). The random number for light transport at this step is \\(\overline{\mathbf{u}}_i^\mathcal{X}\\). Then:

### Solid Angle Perspective

- Jacobian for Vertex Reuse:

<p>
$$
\left|\frac{\partial \omega_i^y}{\partial \omega_i^x}\right| = \left|\frac{\cos \theta_2^y}{\cos \theta_2^x}\right| \cdot \frac{\|\mathbf x_{i+1} - \mathbf x_i\|^2}{\|\mathbf x_{i+1} - \mathbf y_i\|^2}
$$
</p>

where \\(\theta_2^\circ\\) is the angle between \\(\omega_i^\circ\\) and its corresponding surface normal.

- Jacobian for Random Number Reuse:

<p>
$$
\left|\frac{\partial \omega_i^y}{\partial \omega_i^x}\right| = \left|\frac{\partial \omega_i^y}{\partial \overline{\mathbf u}_i^y}\right| \left|\frac{\partial \overline{\mathbf u}_i^y}{\partial \overline{\mathbf u}_i^x}\right|\left|\frac{\partial \overline{\mathbf u}_i^x}{\partial \omega_i^x}\right| = \frac{p_{\omega_i^x}(\mathbf x_{i+1})}{p_{\omega_i^y}(\mathbf y_{i+1})}
$$
</p>

This corresponds to the ratio of probability densities for solid angle sampling.

### Primary Sample Space Perspective

- Jacobian for Random Number Reuse: Always 1.

- Jacobian for Vertex Reuse:

<p>
$$
\left|\frac{\partial \overline{\mathbf u}_i^y}{\partial \overline{\mathbf u}_i^x}\right| = 
\left|\frac{\partial \overline{\mathbf u}_i^y}{\partial \omega_i^y}\right|
\left|\frac{\partial \omega_i^y}{\partial \omega_i^x}\right|
\left|\frac{\partial \omega_i^x}{\partial \overline{\mathbf u}_i^x}\right| =
\frac{p_{\omega_i^y}(\mathbf y_{i+1})}{p_{\omega_i^x}(\mathbf x_{i+1})} \cdot
\left|\frac{\partial \omega_i^y}{\partial \omega_i^x}\right|
$$
</p>

Note that, since the reuse decision is made locally for each step, the global Jacobian is the product of all local Jacobians.


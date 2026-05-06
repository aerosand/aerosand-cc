---
uid: 20251015163537
title: 07_fvmBasics
date: 2025-10-15
update: 2026-03-27
authors:
  - name: Aerosand
    link: https://github.com/aerosand
    image: https://github.com/aerosand.png
tags:
  - cfd
  - cfdb
excludeSearch: false
toc: true
weight: 8
math: true
next:
prev:
comments: true
sidebar:
  exclude: false
draft: false
---


## 0. Preface

The Finite Volume Method (FVM) is based on flux conservation, ensuring that each control volume satisfies physical quantity conservation (which will be gradually understood in subsequent discussions). It is very suitable for numerical computation of fluid dynamics, heat transfer, and related problems. The construction of FVM is relatively simple, and its accuracy generally meets requirements, making it recommended for beginners in CFD.

We have the general form of the fundamental equations of fluid dynamics as follows:

$$
\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) = \nabla\cdot(\Gamma\nabla\phi) + S_{\phi}
$$

At a certain time step, performing volume integration of the fundamental equation over spatially discretized units (grid cells) yields:

$$
\int_{V_P}\bigg(\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) - \nabla\cdot(\Gamma\nabla\phi) \bigg)dV =\int_{V_P} S_{\phi}dV
$$

Rearranged as:

$$
\begin{aligned}
\int_{V_{P}}\bigg(\frac{\partial}{\partial t}(\rho \phi)\bigg)dV + \int_{V_{P}}\bigg(\nabla \cdot (\rho U\phi)\bigg)dV - \int_{V_{P}}\bigg(\nabla\cdot(\Gamma\nabla\phi)\bigg)dV = \int_{V_{P}} S_{\phi}dV \end{aligned}
$$

From left to right, these are the temporal term, convective term, diffusive term, and source term.

The following discussion will focus on the discretization of these four major terms, grasping the main ideas of the finite volume method. Topics such as grids, boundary conditions, and advanced discretization schemes will not be deeply explored for now.

This article mainly discusses:

- [ ] Fundamentals of the Finite Volume Method
- [ ] Understanding the discretization of the four major terms
- [ ] Understanding the correspondence between discretized equations and OpenFOAM code


## 1. Before Discussion

### 1.1. Computational Domain Discretization

Computational domain discretization includes both spatial discretization and temporal discretization.

From the perspective of temporal variables, the governing equations are parabolic equations, so computation can proceed from the initial time step by step until the computational requirements are satisfied.

From the perspective of spatial variables, the governing equations may be hyperbolic or elliptic, requiring different algorithms to solve.

>[!tip]
>The differences between parabolic, hyperbolic, and elliptic equations will not be deeply explored here; they will be gradually understood during learning.

The finite volume method divides the entire spatial computational domain into non-overlapping yet continuous control volumes.

### 1.2. Notation Conventions

For convenience in writing, unless otherwise specified, uppercase letters preferentially represent vector physical quantities based on their physical meaning:

- Velocity vector: $U=(U_{x},U_{y},U_{z})$
- Pressure scalar: $p$
- Any physical quantity to be solved: $\phi$
- Diffusion coefficient: $\Gamma^{\phi}$, related to physical quantity $\phi$, unless otherwise specified, simply written as $\Gamma$
- Source term: $Q^{\phi}$, related to physical quantity $\phi$, unless otherwise specified, simply written as $Q$
- Face vector: $S=(S_{x},S_{y},S_{z})$, always pointing outward

Superscript and subscript notations:

- Superscript $t$ represents the current time step (known quantity), equivalent to superscript $o$, i.e., old time step
- Superscript $t+1$ represents the new time step (quantity to be solved), equivalent to superscript $n$, i.e., new time step
- Superscript $t+n$ represents subsequent time steps, and so on
- Superscript $*$ represents intermediate predicted values based on algorithm iterations
- Subscript $P$ represents the current cell center (using O for Owner is easily confused)
- Subscript $N$ represents the neighboring cell center (Neighbor)
- Subscript $f$ represents the face center between the current cell and neighboring cell, also vaguely representing the cell face

Grid notation:

- Volume $V$, such as the total volume of a cell $V_P$
- Area $\partial{V}$, such as the surface area of a cell $\partial{V_{P}}$
- Cell face vector $S_{f}$, with area magnitude $|S_{f}|$

For internal faces, the face vector points from the control volume with smaller index (owner) to the control volume with larger index (neighbor). For boundary faces, the face vector always points outward.

For a given control volume, when traversing all its control faces:

- If the neighboring cell index is larger than the current cell index, then the face vector between them points from the current cell to the neighboring cell
  - For the current control volume, this face vector points outward, consistent with the mathematical convention for integrals and fluxes, so it is positive
  - For this face, the cell with the smaller index is stored in the owner array, and the cell with the larger index is stored in the neighbor array
- If the neighboring cell index is smaller than the current cell index, then the face vector between them points from the neighboring cell to the current cell
  - For the current control volume, this face vector points inward, opposite to the mathematical convention for integrals and fluxes, so it is negative
  - For this face, the cell with the smaller index is still stored in the owner array, and the cell with the larger index is stored in the neighbor array
- For the owner array of faces, the natural index is the face index, and the element value is the index of the owner cell for that face; similarly for the neighbor array
- When computing a cell and traversing to a face, if indexing reveals that the cell is the owner of that face, then in the computation for this cell, the face vector is positive; if the cell is the neighbor of that face, then the face vector is negative

>[!tip]
>When discussing indices later, you can revisit this discussion about indices.

### 1.3. Control Volume

A control volume is a small volume unit obtained by dividing the continuous computational domain in the finite volume method, generally corresponding to a grid cell in mesh generation.

We define the center through volume: the center point vector $\vec{x}_{P}$ multiplied by volume $V_{P}$ equals the integral of the position vector $\vec{x}$ over the volume within that space, i.e.,

$$
V_{P}\vec x_{P} = \int_{V_{P}}\vec{x} dV
$$

Replacing volume with volume integral, rearranging gives:

$$
\int_{V_{P}}\vec{x}dV - \vec{x}_P\int_{V_{P}}dV = 0
$$

Further rearranging:

$$
\int_{V_{P}}(\vec{x} - \vec{x}_{P})dV= 0
$$

The above equation defines the center point $\vec{x}_{P}$ of a spatial volume.

### 1.4. Discretization Accuracy

For a continuous field $\phi$ within it, the discretized field can always be represented using Taylor expansion.

Spatially:

$$
\phi(\vec{x}) \approx \phi_{P}+ (\vec{x} - \vec{x}_{P})\cdot(\nabla\phi)_{P}
$$

Temporally:

$$
\phi^{t+\Delta t} \approx \phi^{t} + \Delta t(\frac{\partial \phi}{\partial t})^{t}
$$

The above two discretization schemes have second-order accuracy, which generally satisfies the requirements for solving partial differential equations in computational fluid dynamics.

### 1.5. Function Integration

Based on the second-order Taylor approximation and the definition of the control volume center point, continuous functions are handled as follows:

$$
\begin{aligned}
\int_{V_{P}}\phi(\vec{x})dV &\approx \int_{V_{P}}\bigg[\phi_{P}+ (\vec{x} - \vec{x}_{P})\cdot(\nabla\phi)_{P}\bigg]dV \\
&= \phi_{P}\int_{V_{P}}dV + \bigg[\int_{V_{P}}(\vec{x} - \vec{x}_{P})dV\bigg]\cdot(\nabla\phi)_{P} \\
&= \phi_{P}V_{P}
\end{aligned}
$$

Rearranging:

$$
\int_{V_{P}}\phi(\vec{x})dV = \phi_{P}V_{P}
$$

That is, the integral of a continuous function over a volume space approximately equals the function value at the center point of that volume space multiplied by the volume.

### 1.6. Divergence Integration

We know Gauss's theorem states:

For vectors:

$$
\int_{V}\nabla\cdot\vec{a}dV = \int_{\partial V}\vec{a}\cdot d\vec{S}
$$

For scalars:

$$
\int_V\nabla\phi dV = \int_{\partial V}\phi d\vec{S}
$$

Applying Gauss's theorem, we convert volume integrals to surface integrals (no approximation):

The volume integral of vector field divergence computation, using Gauss's theorem:

$$
\begin{aligned}
\int_{V_{P}}\nabla\cdot\vec{a}dV = \int_{\partial V_{P}}\vec{a}\cdot d\vec{S}
\end{aligned}
$$

The volume integral of physical quantity divergence computation essentially states that the flux integral over the volume boundary equals the divergence integral within the volume.

Surface area discretization (no approximation):

$$
\int_{\partial V_{P}}\vec{a}\cdot d\vec{S} = \sum\limits_{f}(\int_f\vec{a}\cdot d\vec{S})
$$

Further discretized as the sum of area integrals computed from physical quantities and face vectors on each discrete face.

Numerical integration approximation (approximation):

Numerical quadrature of integrals essentially replaces integration with summation, also known as mechanical quadrature. The integrand is evaluated at multiple discrete points, then summed.

Specifically for this computation:

$$
\int_{f}\vec{a}\cdot d\vec{S} \approx  \sum\limits_{ip}\omega_{ip}(\vec{a}_{f}\cdot \vec{S}_f)_{ip}
$$

where $\omega_{ip}$ are integration nodes. The essence of this approximation is that continuous integral computation approximates the cumulative weighted algebraic computation at integration nodes. When cells are sufficiently small, and discrete faces are also sufficiently small, taking the center point of the discrete face as the only integration node can also satisfy accuracy requirements.

At this point, the above equation approximately becomes:

$$
\int_{f}\vec{a}\cdot d\vec{S} \approx  \vec{a}_{f}\cdot \vec{S}_{f}
$$

Substituting back into divergence computation gives (although approximate, for convenience we'll write equality directly):

$$
\int_{V_{P}}\nabla\cdot\vec{a}dV = \sum\limits_{f}\vec{a}_{f}\cdot \vec{S}_{f}
$$

On sufficiently small control volumes, divergence integrals are independent of volume and can be treated as independent continuous functions. Continuous functions can be handled as described above, ultimately giving:

$$
(\nabla\cdot\vec{a})V_{P}= \sum\limits_{f}\vec{a}_{f}\cdot \vec{S}_{f}
$$

Note again our convention: $S_{f}$ is positive only when the current cell is the owner of that face, pointing outward from point P to point N. When the current cell is the neighbor of that face, $S_{f}$ points inward from point N to point P, opposite to the mathematical definition, and is negative. Therefore, the above equation can actually be written as:

$$
\sum\limits_{f}\vec{a}_{f}\cdot \vec{S}_{f}= \sum\limits_{owner}\vec{a}_{f}\cdot \vec{S}_{f} - \sum\limits_{neighbor}\vec{a}_{f}\cdot \vec{S}_{f}
$$


## 2. Convective Term

The spatial integral of the convective term is:

$$
\int_{V_{P}}\nabla\cdot(\rho U\phi)dV
$$

### 2.1. Convective Discretization

The convective term discretization process is:

$$
\begin{aligned}
\int_{V_{P}}\nabla\cdot(\rho U\phi)dV &= \int_{\partial V_{P}}(\rho U\phi)\cdot d S \\
&= \sum\limits_{f}\int_{f}(\rho U\phi) \cdot d S\\
&\approx \sum\limits_{f}(\rho U\phi)_{f}\cdot S_{f} =  \sum\limits_{f}F^{C}_{f}\cdot S_{f} \\
&= \sum\limits_{f}(\rho U_{f}\cdot S_{f})\phi_{f} =  \sum\limits_{f}F_{f}\phi_{f}
\end{aligned}
$$

>[!tip]
>The discussion has just begun. To make reading smoother and clearer, some content will be mentioned repeatedly; we'll skip appropriately later.

The detailed analysis of the discretization computation process is as follows:

Volume integral to surface integral conversion (no approximation):

$$
\int_{V_{P}}\nabla\cdot(\rho U\phi)dV = \int_{\partial V_{P}}(\rho U\phi)\cdot dS
$$

Surface area discretization (no approximation):

$$
\int_{\partial V_{P}}(\rho U\phi)\cdot d S = \sum\limits_{f}\int_{f}(\rho U\phi) \cdot dS
$$

- The cell surface is a complete closed surface composed of several discrete faces, e.g., a hexahedral mesh consists of six discrete faces.
- The surface integral equals the sum of integrals over these six discrete faces (no approximation).

For the surface integral on a single discrete face, numerical quadrature gives (approximation):

$$
\int_{f}(\rho U\phi)_{f} \cdot dS \approx \sum\limits_{ip}\omega_{ip}(\rho U\phi)_{ip}\cdot S_{ip}
$$

For numerical computation grids, when grids are very small, a single-point approximation suffices:

$$
\int _{f}(\rho U\phi)_{f}\cdot dS \approx (\rho U\phi)_{f}\cdot S_{f} = \sum\limits_{f}(\rho U_{f}\cdot S_{f})\phi_{f}
$$

Note that this point is taken at the center of the discrete face.

Combining the above discussion, the semi-discrete approximation of the convective term ultimately becomes:

$$
\int_{V_{P}}\nabla\cdot (\rho U\phi)dV  \approx \sum\limits_{f}(\rho U\phi)_{f}\cdot S_{f} =  \sum\limits_{f}F^{C}_{f}\cdot S_{f} = \sum\limits_{f}F_{f}\phi_{f}
$$

where $F^{C}$ is called the convective flux (vector), reflecting the physical essence of convection:

$$
F^{C}_{f} = (\rho U\phi)_{f}
$$

If $\phi=U$, then:

$$
F^{C}_{f} = (\rho U U)_{f}
$$

Here $F^{C}$ is also called momentum flux.

And $F_{f}$ is called mass flux (scalar):

$$
F_{f} = \rho U_{f}\cdot S_{f}
$$

Mass flux reflects the more fundamental physical conservation essence during convection.

>[!tip]
>Some may have questions about the position of the dot product with the face vector. Actually, vector dot product satisfies commutativity, so the position can be before or after.

Overall, the convective term represents the change of physical quantity with mass transport, mathematically manifested as the physical quantity multiplied by mass flux across the cross-section.

### 2.2. Convective Interpolation

Note that after discretization of the convective term, values like $(\rho U)_{f}$ are face center values. For the finite volume method, all values are control volume center values, so we need to interpolate from cell center values to face center values.

We can use the two cell center values closest to the face for interpolation.

**Central Differencing (CD)**

$$
\phi_{f}= f_{x}\phi_{P}+ (1-f_{x})\phi_{N}
$$

The interpolation factor is:

$$
f_{x}= \frac{|fN|}{|PN|}
$$

- Literature has proven that CD scheme is second-order accurate, even for non-uniform grids.
- May produce non-physical oscillations in convection-dominated flows.
- May violate solution boundedness.

>[!tip]
>It's okay if you don't understand yet; confusion will gradually decrease as the discussion progresses.

**Upwind Differencing (UD)**

$$
\phi_{f}= f_{x}\phi_{P}+ (1-f_{x})\phi_{N}
$$

where the coefficients are:

$$
f_{x} =
\begin{cases}
1, \phi_{f} =  \phi_{P},  \space for F \geq  0 \\
0, \phi_{f} =  \phi_{N},  \space for F  \leq 0
\end{cases}
$$

That is, the face center value always takes the upstream cell center value.

- Literature has proven UD is unconditionally bounded and stable.
- Does not produce non-physical oscillations.
- Only first-order accurate; solutions may have significant deviation.

**Blended Differencing (BD)**

To simultaneously ensure accuracy and boundedness, a linear combination of CD and UD schemes can be constructed:

$$
\phi_{f}= (1-\gamma)(\phi_{f})_{UD} + \gamma(\phi_{f})_{CD}
$$

where $(\phi_{f})_{UD}$ is the value based on UD scheme, $(\phi_{f})_{CD}$ is the value based on CD scheme. $\gamma$ is the blending factor, with value $0 \leq \gamma\leq 1$.

Or written as a whole:

$$
\begin{aligned}
\phi_{f} &= \bigg[(1-\gamma)\max(sgn(F_{f}),0) + \gamma f_{x}\bigg]\phi_{P} \\
&+ \bigg[(1-\gamma)\min(sgn(F_{f}),0) + \gamma(1-f_{x})\bigg]\phi_{N}
\end{aligned}
$$

- Literature proves BD scheme can effectively ensure both accuracy and boundedness.

OpenFOAM provides the corresponding `Gauss upwind` scheme. In OpenFOAM's `fvSchemes`, you can specify:

```cpp
divSchemes
{
	default Gauss upwind;
}
```

Here, `Gauss` indicates discretization is based on Gauss's theorem, `upwind` indicates interpolation uses upwind scheme. OpenFOAM provides other options, which we won't explore deeply for now.


## 3. Diffusive Term

The volume integral of the diffusive term is:

$$
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV
$$

### 3.1. Diffusive Discretization

$$
\begin{align*}
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV &= -\int_{\partial V_{P}}(\Gamma\nabla \phi)\cdot dS\\
&= -\sum\limits_{f}\int_{f}(\Gamma\nabla \phi)\cdot dS\\
&\approx -\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f}\\
&= \sum\limits_{f}F^{D}_{f}\cdot S_{f}
\end{align*}
$$

The detailed analysis of the discretization computation process is as follows:

Volume integral to surface integral conversion (no approximation):

$$
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV = -\int_{\partial V_{P}}(\Gamma\nabla \phi)\cdot dS
$$

Surface area discretization (no approximation):

$$
-\int_{\partial V_{P}}(\Gamma\nabla \phi)\cdot dS = -\sum\limits_{f}\int_{f}(\Gamma\nabla \phi)\cdot dS
$$

- The cell surface is a complete closed surface composed of several discrete faces, e.g., a hexahedral mesh consists of six discrete faces.
- The surface integral equals the sum of integrals over these six discrete faces (no approximation).

Numerical integration (approximation):

For the surface integral on a single discrete face, numerical quadrature gives:

$$
-\int_{f}(\rho\Gamma\nabla \phi)\cdot dS \approx -\sum\limits_{ip}\omega_{ip}(\rho\Gamma_{\phi}\nabla \phi)_{ip}\cdot S_{ip}
$$

For numerical computation grids, when grids are very small, a single-point approximation suffices:

$$
-\int_{f}(\Gamma\nabla \phi)\cdot dS \approx -(\rho\Gamma\nabla \phi)_{f}\cdot S_{f}
$$

Note that this point is taken at the center of the discrete face.

Combining the above discussion, the semi-discrete approximation of the diffusive term ultimately becomes:

$$
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV \approx -\sum_{f}(\Gamma\nabla \phi)_f\cdot S_{f} = \sum\limits_{f}F^{D}_{f}\cdot S_{f}
$$

where $F^{D}$ is called diffusive flux:

$$
F^{D}_{f} = (-\Gamma\nabla\phi)_{f}
$$

>[!tip]
>Understanding the negative sign:
>1. For the general equation, except for the source term, the temporal, convective, and diffusive terms are placed on the left side, mathematically forming flux accumulation.
>2. Physically, flux direction is opposite to gradient direction.

### 3.2. Gradient Interpolation

For orthogonal grids, gradient can be computed along the face vector direction $\vec{n}$ and the direction perpendicular to the face vector $\vec{\tau}$:

$$(\nabla \phi)_{f} = (\frac{\partial \phi}{\partial \vec{n}})_{f} + (\frac{\partial \phi}{\partial \vec{\tau}})_{f}$$

Dot product with face vector, the perpendicular component becomes zero, leaving only the normal direction component:

$$
(\nabla \phi)_{f}\cdot S_{f} = (\frac{\partial \phi}{\partial \vec{n}})_{f}|S_{f}|
$$

The remaining gradient computation can be interpolated by distance:

$$
(\frac{\partial \phi}{\partial \vec{n}})_{f} = \frac{\phi_{N} - \phi_{P}}{|\vec{d}|}
$$

That is:

$$
(\nabla \phi)_{f} \cdot S_{f} =\frac{\phi_{N} - \phi_{P}}{|\vec{d}|} |S_{f}|
$$

Alternatively, gradient computation can be linearly interpolated:

$$
(\nabla \phi)_{f}= f_x(\nabla \phi)_{P}+(1-f_x)(\nabla \phi)_N
$$

The interpolation coefficient can refer to the CD scheme coefficient mentioned earlier.

These methods all convert face center gradient computation to linear interpolation of cell center gradient computation.

### 3.3. Gradient Computation

If using the linear interpolation above, we still need to compute cell center gradient:

$$
(\nabla \phi)_{P}
$$

If performing volume integration, using Gauss's theorem, surface area discretization, and numerical integration:

$$
\int_{V_{P}}(\nabla \phi)_{P}dV = \int_{\partial V_{P}}\phi dS = \sum\limits_{f}\int_{f}\phi dS \approx \sum\limits_{f}\phi_{f}S_{f}
$$

And the volume integral equals:

$$
\int_{V_{P}}(\nabla \phi)_{P}dV = V_{P}(\nabla \phi)_{P}
$$

Thus we obtain:

$$
(\nabla \phi)_{P} = \frac{1}{V_{P}}\sum\limits_{f}\phi_{f}S_{f}
$$

The face value $\phi_{f}$ in the above equation can be obtained through linear interpolation:

$$
\phi_{f} = (1-\lambda)\phi_{P} + \lambda\phi_{N}
$$

OpenFOAM provides the corresponding `Gauss linear` scheme. In OpenFOAM's `fvSchemes`, you can specify:

```cpp
gradSchemes
{
	default Gauss linear;
}
```

Here, `Gauss` indicates discretization is based on Gauss's theorem, `linear` indicates interpolation uses linear interpolation scheme, with coefficients using cell center distance ratio. OpenFOAM provides other options, which we won't explore deeply for now.

### 3.4. Non-orthogonal Correction

The face vector $S_{f}$ of a non-orthogonal grid can be decomposed as:

$$\vec S_f = \vec \Delta_f + \vec k_f$$

Thus:

$$
(\nabla \phi)_{f}\cdot S_{f} = (\nabla \phi)_{f}\cdot\vec{\Delta} + (\nabla \phi)_{f}\cdot\vec{k}
$$

In the decomposition of face vector, we choose $\vec{\Delta}$ to be parallel to $\vec{d}$ (the direction vector from current cell center to neighbor cell center), so the orthogonal influence term can continue to use the method of 【orthogonal grid gradient computation】. Deviation caused by grid non-orthogonality is corrected by the non-orthogonal correction term.

Face vector decomposition can use the following methods:

**Minimum Correction Method**

Construct decomposition with $\vec{S}$ as hypotenuse, $\vec{\Delta}$ and $\vec{k}$ as right-angle sides.

Then:

$$
\vec{\Delta} = \frac{\vec{d}\cdot\vec{S}}{\vec{d}\cdot\vec{d}}\vec{d}
$$

The corresponding $\vec{k}$ is decomposed from the face vector.

With this method, as grid non-orthogonality increases, more of the face vector decomposes into $\vec{k}$, meaning the influence of $\phi_P$ and $\phi_N$ decreases.

**Orthogonal Correction Method**

Construct decomposition with $\vec{S}$ and $\vec{\Delta}$ as equal-length sides, $\vec{k}$ as the third side.

Then:

$$
\vec{\Delta} = \frac{\vec{d}}{|\vec{d}|}|\vec{S}|
$$

With this method, the influence of $\phi_P$ and $\phi_N$ does not change with non-orthogonality.

**Over-relaxation Method**

Construct decomposition with $\vec{S}$ and $\vec{k}$ as right-angle sides, $\vec{\Delta}$ as hypotenuse.

Then:

$$
\vec{\Delta} = \frac{\vec{d}}{\vec{d}\cdot\vec{S}}|\vec{S}|^2
$$

With this method, as grid non-orthogonality increases, more of the face vector decomposes into $\vec{\Delta}$, meaning the influence of $\phi_P$ and $\phi_N$ increases.

For surface normal gradient, OpenFOAM provides the corresponding `orthogonal` scheme. In OpenFOAM's `fvSchemes`, you can specify:

```cpp
snGradSchemes
{
	default orthogonal;
}
```

Here, `orthogonal` assumes orthogonal grids, using the simplest difference scheme. OpenFOAM provides other options, which we won't explore deeply for now.


### 3.5. Final Form

For the orthogonal influence part, if interpolated by distance (for simplicity):

$$
(\nabla \phi)_{f}\cdot\vec{\Delta} = \frac{\phi_{N}- \phi_{P}}{|\vec{d}|}\cdot \vec{\Delta}
$$

The final form is:

$$
(\nabla \phi)_{f}\cdot S_{f} = \frac{\phi_{N}- \phi_{P}}{|\vec{d}|}\cdot \vec{\Delta} + (\nabla \phi)_{f}\cdot\vec{k}
$$

The gradient computation on the face in the second term on RHS can refer to earlier discussion, obtained by volume interpolation.

Combining with previous diffusive term rearrangement result:

$$
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV \approx -\sum_{f}(\Gamma\nabla \phi)_f\cdot S_{f} = \sum\limits_{f}F^{D}_{f}\cdot S_{f}
$$

Substituting the computed diffusive term gives:

$$
\begin{align*}
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV &=  -\int_{\partial V_{P}}(\Gamma\nabla \phi)\cdot dS \\
&\approx -\sum_{f}(\Gamma\nabla \phi)_f\cdot S_{f} \\
&=  -\sum_{f}\Gamma_{f}\bigg[\frac{\phi_{N} - \phi_{P}}{|\vec{d}|}\cdot\vec{\Delta} + (\nabla \phi)_f\cdot\vec k\bigg]
\end{align*}
$$

If diffusion coefficient is uniform, it can be directly computed. If diffusion coefficient is not uniform, similar interpolation approximation computation is needed:

$$
\Gamma_{f} = (1 - \lambda)\Gamma_{P}+ \lambda \Gamma_{N}
$$

OpenFOAM provides the corresponding `Gauss linear orthogonal` scheme. In OpenFOAM's `fvSchemes`, you can specify:

```cpp
laplacianSchemes
{
	default Gauss linear orthogonal;
}
```

Here, `Gauss` indicates discretization is based on Gauss's theorem, `linear` indicates face gradient interpolation uses linear scheme, i.e.,

$$
(\nabla \phi)_{f}= f_x(\nabla \phi)_{P}+(1-f_x)(\nabla \phi)_N
$$

And `orthogonal` indicates only orthogonal influence is considered, ignoring non-orthogonal correction. OpenFOAM provides other options, which we won't explore deeply for now.

## 4. Source Term

Broadly speaking, all terms that cannot be classified as convective, diffusive, or temporal terms are treated as source terms.

Generally, we first consider whether the source term can be linearized. If linearizable:

$$
S_{\phi}(\phi) = Su + Sp\phi
$$

Spatial integral:

$$
\int_{V_{P}}S_{\phi}(\phi)dV = SuV_{P}+ SpV_{P}\phi_{P}
$$

Let's particularly discuss the pressure source term. If pressure term is explicitly discretized, using pressure from previous known step:

$$
\begin{align*}
\int_{V_{P}}\nabla pdV &= \int_{\partial V_{P}}pdS \\
&= \sum_{f}p_{f}S_{f} \\
&= \sum_{f}p_{f}^{t}S_{f}
\end{align*}
$$

Note that $S_{f}$ above is the face vector according to convention.

When handling source terms, note that linearization may weaken diagonal dominance of the LHS coefficient matrix after rearrangement, affecting computation stability. Therefore, source term handling should pay attention to diagonal dominance of the LHS coefficient matrix.

## 5. Temporal Term

The volume integral of the temporal term is:

$$\int_{V_{P}}\bigg(\frac{\partial}{\partial t}(\rho \phi)\bigg)dVdt$$

The partial derivative with respect to time has no relation to volume, so rearranging gives:

$$
\int_{V_{P}}\bigg(\frac{\partial}{\partial t}(\rho \phi)\bigg)dVdt = \frac{\partial}{\partial t}(\rho \phi)\int_{V_{P}}dV = V_{P}\frac{\partial \rho \phi}{\partial t}
$$

The partial derivative with respect to time requires further processing.

## 6. Temporal Discretization

Combining previous discussions on spatial discretization, we further discuss temporal discretization.

Performing time integration of the fundamental equation over one time step:

$$
\begin{aligned}
\int_{t}^{t+1} \bigg[\int_{V_P}\bigg(\frac{\partial}{\partial t}(\rho \phi)\bigg)dV + \int_{V_P}\bigg(\nabla \cdot (\rho U\phi)\bigg)dV &- \int_{V_P}\bigg(\nabla\cdot(\Gamma\nabla\phi)\bigg)dV \bigg]dt\\
&= \int_{t}^{t+1}\bigg[\int_{V_P} S_{\phi}dV \bigg]dt
\end{aligned}
$$

Substituting previous discussions, assuming source term can be linearized:

$$\begin{aligned}
\int_{t}^{t+1}\bigg[\frac{\partial \rho\phi}{\partial t}V_{P}+\sum\limits_{f}F^{C}_{f}\cdot S_{f}&+ \sum\limits_{f}F^{D}_{f}\cdot S_{f} \bigg]dt\\
&= \int_{t}^{t+1}\bigg[SuV_{P}+ SpV_P\phi_{P}\bigg]dt
\end{aligned}
$$

Further substitution:

$$\begin{aligned}
\int_{t}^{t+1}\bigg[\frac{\partial \rho\phi}{\partial t}V_{P}+\sum\limits_{f}(\rho U\phi)_{f}\cdot S_{f}&-\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f} \bigg]dt\\
&= \int_{t}^{t+1}\bigg[SuV_{P}+ SpV_P\phi_{P}\bigg]dt
\end{aligned}
$$

This expression is generally called the semi-discrete equation.

Considering continuous field changes over time:

Temporal term:

$$
\bigg(\frac{\partial \rho \phi}{\partial t}\bigg)_{P} = \frac{\rho^{t+1}_{P}\phi^{t+1}_{P}-\rho^{t}_{P}\phi^{t}_{P}}{\Delta t}
$$

For time integral of continuous function:

$$
\int_{t}^{t+1}\phi(t)dt \approx ( (1-f)\phi^{t+1} + f\phi^{t} )\Delta t
$$

Coefficient $f$ taking different values corresponds to different discretization schemes:

$$
\begin{aligned}
\frac{\rho^{t+1}_{P}\phi^{t+1}_{P}-\rho^{t}_{P}\phi^{t}_{P}}{\Delta t} V_{P}&+\\
\sum\limits_{f}\bigg[(1-f)F_{f}^{C(t+1)}\cdot S_{f} + fF^{C(t)}_{f}\cdot S_{f} \bigg] &- \sum\limits_{f} \bigg[(1-f)F_{f}^{D(t+1)}\cdot S_{f}+fF_{f}^{D(t)}\cdot S_{f} \bigg]\\
&=  SuV_{P}+(1-f)S_pV_P\phi_{P}^{t+1} + fS_pV_P\phi_{P}^{t}
\end{aligned}
$$

### 6.1. Euler Scheme

When coefficient $f=0$, this scheme is called Euler scheme in OpenFOAM.

If convective and diffusive terms use `fvm` for implicit discretization, the discrete equation becomes:

$$
\begin{aligned}
\frac{\rho^{t+1}_{P}\phi^{t+1}_{P}-\rho^{t}_{P}\phi^{t}_{P}}{\Delta t} V_{P} &+\\ \sum\limits_{f}F^{C(t+1)}_{f}\cdot S_{f}&- \sum\limits_{f} F_{f}^{D(t+1)}\cdot S_{f}\\
&=  SuV_{P} + S_{p}V_{P}\phi_{P}^{t+1}
\end{aligned}
$$

According to previous discussion, $F_{f}^{C},F_{f}^{D}$ can be interpolated as combinations of current cell (subscript $P$) and neighbor cell (subscript $N$) center value computations.

Placing all unknowns (new time step) on LHS, known quantities (old time step) and source terms on RHS, finally organized into linear algebraic system for solving:

$$
a_{P}\phi_{P}^{n}+\sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P}
$$

This discretization scheme has first-order accuracy in time and second-order accuracy in space, with relatively simple numerical computation, stable for small time steps.

OpenFOAM provides the corresponding `Euler` time scheme, specified in `fvSchemes` as:

```cpp
ddtSchemes
{
	default    Euler;
}
```

### 6.2. CN Scheme

When coefficient takes a certain value (e.g., 0.5), this scheme is called semi-implicit Crank-Nicolson scheme in OpenFOAM.

If convective and diffusive terms use `fvm` for implicit discretization, the discrete equation becomes:

$$
\begin{aligned}
\frac{\rho^{t+1}_{P}\phi^{t+1}_{P}-\rho^{t}_{P}\phi^{t}_{P}}{\Delta t} V_{P}&+\\
\sum\limits_{f}\bigg[\frac{1}{2}F_{f}^{C(t+1)}\cdot S_{f} + \frac{1}{2}F^{C(t)}_{f}\cdot S_{f} \bigg] &- \sum\limits_{f} \bigg[\frac{1}{2}F_{f}^{D(t+1)}\cdot S_{f}+ \frac{1}{2}F_{f}^{D(t)}\cdot S_{f} \bigg]\\
&=  SuV_{P}+ \frac{1}{2}S_pV_P\phi_{P}^{t+1} + \frac{1}{2}S_pV_P\phi_{P}^{t}
\end{aligned}
$$

We can see that the CN scheme also performs time-weighted averaging on spatial terms (convective, diffusive, source terms).

Similarly, placing all unknowns (new time step) on LHS, known quantities (old time step) and source terms on RHS, finally organized into linear algebraic system for solving:

$$
Ax = b
$$

Mathematical verification shows this CN scheme has second-order accuracy in both time and space.

OpenFOAM provides the corresponding `CrankNicolson` time scheme. In OpenFOAM's `fvSchemes`, you can specify:

```cpp
ddtSchemes
{
	default    CrankNicolson 0.5;
}
```

### 6.3. Backward Scheme

This backward scheme is still fully implicit but improves temporal term discretization accuracy.

If convective and diffusive terms use `fvm` for implicit discretization, the discrete equation becomes:

$$
\begin{aligned}
\frac{\frac{3}{2}\rho^{t+1}_{P}\phi^{t+1}_{P}-2\rho^{t}_{P}\phi^{t}_{P}+ \frac{1}{2}\rho^{t-1}_{P}\phi^{t-1}_{P}}{\Delta t} V_{P} &+\\ \sum\limits_{f}F^{C(t+1)}_{f}\cdot S_{f}&- \sum\limits_{f} F_{f}^{D(t+1)}\cdot S_{f}\\
&=  SuV_{P} + S_{p}V_{P}\phi_{P}^{t+1}
\end{aligned}
$$

We can see that this scheme's temporal term uses three time layers, while spatial terms (convective, diffusive, source terms) completely use new time step values.

Similarly, discrete equations can be organized into linear algebraic system for solving.

Mathematical verification shows this scheme has second-order accuracy in both time and space.

OpenFOAM provides the corresponding `backward` time scheme. In OpenFOAM's `fvSchemes`, you can specify:

```cpp
ddtSchemes
{
	default    backward;
}
```


## 7. Boundary Conditions

We discuss numerical boundary conditions and physical boundary conditions separately.

### 7.1. Numerical Boundary Conditions

There are two basic types of numerical boundary conditions:

- Dirichlet (fixed value of variable at the boundary)
- Neumann (fixed gradient of variable normal to the boundary)

Additionally, different mixed forms of these basic types may appear.

These types need to be applied to the algebraic system before solving the linear algebraic system.

**Non-orthogonality**

Before formally applying numerical boundary conditions to the linear algebraic system, we need to examine the influence of grid non-orthogonality.

For a boundary control volume, the control volume center point is still $P$, the boundary face center point is $b$, the vector from volume center to boundary face center is $\vec d$. Due to non-orthogonality, the normal vector from volume center to boundary face is $\vec{d}_{n}$.

If we still decompose face vector $\vec{S}$ as $\vec{S} = \vec{\Delta} + \vec{k}$, since no neighbor cell exists outside the boundary face, we simply consider $\vec{\Delta}$ identical to $\vec{S}$.

$$
\vec{d}_{n}= \frac{\vec{S}}{|\vec{S}|}\frac{\vec{d}\cdot{S}}{|\vec{S}|}
$$

At this point, we no longer use the non-orthogonal correction term $\vec{k}$.

We can also continue using the non-orthogonal correction term to improve discretization accuracy.

**Fixed Value BC**

That is, on the boundary face:

$$
\phi_f = \phi_b
$$

Both convective and diffusive terms handle this value.

For convective term:

$$
\int_{V_{P}}\nabla\cdot(\rho U\phi)dV = \sum\limits_fF_{f}\phi_{f}
$$

So when face summation traverses to the boundary face, it should be directly replaced with fixed value:

$$
F_{f}\phi_{f} = F_b\phi_b
$$

For diffusive term:

$$
-\int_{V_{P}}\nabla\cdot(\Gamma \nabla\phi)dV = -\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f}
$$

When computing $(\nabla \phi)_{f}\cdot S_{f}$, fixed value also needs to be used:

$$\begin{aligned}
(\nabla \phi)_{f}\cdot S_{f} &= |\vec{\Delta}|\frac{\phi_{N}- \phi_{P}}{|\vec{d}|} + \cancel{\vec{k}\cdot(\nabla\phi)_{f}} \\
&= |\vec{S}|\frac{\phi_{b}- \phi_{P}}{|\vec{d}_{n}|}
\end{aligned}$$

**Fixed Gradient BC**

That is, on the boundary:

$$
\bigg(\frac{\vec S}{|\vec S|}\cdot \nabla \phi\bigg)_{b} = g_{b}
$$

For convective term:

Gradient can be inversely computed:

$$\begin{aligned}
\phi_{b} &= \phi_{P} + \vec d_{n}\cdot (\nabla\phi) \\
&= \phi_{P} + |\vec d_{n}|g_{b}
\end{aligned}$$

Obtain fixed value then substitute.

Note, due to grid non-orthogonality, $\vec{d}_{n}$ does not completely point to the boundary face center point, so Fixed Gradient BC computation is first-order accurate. We can add non-orthogonal correction terms to improve accuracy.

For diffusive term:

Face gradient directly substituted for computation:

$$
-\int_{V_{P}}\nabla\cdot(\Gamma \nabla\phi)dV = -\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f}
$$

When face summation traverses to the boundary face:

$$
\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f}= \Gamma_{b}|S|g_b
$$

### 7.2. Physical Boundary Conditions

**Incompressible Flow**

- Inlet BC
	- velocity: fixed value
	- pressure: fixed gradient zero
- Outlet BC
	- Velocity can be mapped from adjacent cells and scaled to satisfy overall continuity
		- Pressure specified as zero gradient
		- May cause instability
	- Velocity can be specified as zero gradient
		- Pressure uses fixed value
		- Overall mass conservation ensured through pressure equation
- Symmetry plane BC
	- Normal gradients on boundary are all zero
- Impermeable no-slip walls
	- Fluid velocity on wall equals wall velocity itself
	- Achieved by fixing wall flux as zero
	- Pressure gradient zero

**Compressible Flow**

The basic idea is the same, but transonic and supersonic cases are special.

Turbulence boundary conditions are also special; these require consulting specialized literature.

## 8. Algebraic System

Regardless, discrete equations can finally be organized as:

$$
a_P\phi_{P}^{n}+\sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P}
$$

- Convective term considers CD / UD / BD interpolation schemes
- Diffusive term considers gradient interpolation computation and non-orthogonal correction
- Source term linearization

For each control volume, there will be such a locally assembled discrete equation, forming a linear algebraic system:

$$
[A]\bigg[\phi\bigg] = [R]
$$

- Matrix $[A]$ is sparse, $a_P$ on diagonal, $a_N$ beside diagonal.
- Matrix $\bigg[\phi\bigg]$ is a vector formed by all cell center values to be solved in the computational domain
- Matrix $[R]$ is the source term vector, containing all terms without $\phi$, all terms with $\phi$ from old time step
- Coefficient $a_P$ partly comes from temporal term, partly from convective and diffusive terms, partly from linear part of source term
- Coefficient $a_N$ partly comes from neighbor cell parts involved in each term computation

When this linear algebraic system is solved, we obtain the physical field $\phi$ (grid cell center values) at the new time step.

Existing algebraic solution methods are divided into direct methods and iterative methods. Generally, when matrix size increases, iterative methods are more economical but also have some requirements on matrix characteristics.

### 8.1. Matrix Properties

The above LHS matrix is sparse; most matrix elements equal zero. If solvers can utilize sparse characteristics, computer memory requirements can be greatly reduced.

When diagonal elements equal the sum of non-diagonal elements, it's called diagonally equal. When diagonal elements are greater than the sum of non-diagonal elements, it's called diagonal dominance. A matrix with at least one row satisfying diagonal dominance is called a diagonally dominant matrix.

Iterative methods require matrix diagonal dominance to ensure convergence.

Source term discretization directly relates to diagonal dominance.

Recall RHS source term linearization:

$$
S_{\phi}(\phi) = Su + Sp\phi
$$

If $S_{p} < 0$, when moved to LHS, it increases diagonal dominance of LHS matrix, thereby increasing iterative computation convergence.

If $S_{p}> 0$, when moved to LHS, it decreases diagonal dominance of LHS matrix, weakening computation convergence. In this case, it should be explicitly discretized, kept on RHS.

Convective term discretization schemes can all be seen as upgrades of UD scheme. During discretization, one can optionally perform implicit discretization (kept in LHS matrix) or explicit discretization (kept in RHS matrix).

Diffusive term discretization generates diagonally equal matrices on orthogonal grids, but grid non-orthogonality breaks this, also breaking boundedness. Generally, the non-orthogonal correction part of diffusive term is small compared to the implicit part (orthogonal influence part, discretized to LHS). Therefore, the orthogonal influence part is implicitly handled, kept in RHS, generating diagonally equal matrix. The non-orthogonal correction part is explicitly handled, moved to RHS, added to generalized source term. If non-orthogonal influence is large, non-orthogonal correctors need to be added, using additional iterations to compute non-orthogonal correction term, updating non-orthogonal correction term each iteration until error meets requirements. These separate implicit-explicit treatments can only improve matrix properties, not guarantee boundedness.

### 8.2. Under-relaxation

Practice proves that the linear part of source term and temporal term can actually strengthen diagonal dominance properties of LHS matrix.

In steady-state computations, without temporal term discretization strengthening diagonal dominance of LHS matrix, under-relaxation technique can be used.

Consider the linear algebraic system obtained after discretization:

$$
a_P\phi_{P}^{n}+\sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P}
$$

Add artificial term to this equation:

$$
a_P\phi_{P}^{n}+ \bigg(\frac{1-\alpha}{\alpha}a_P\phi_{P}^{n}\bigg) + \sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P} + \bigg(\frac{1-\alpha}{\alpha}a_P\phi_{P}^{0}\bigg)
$$

- $0<\alpha\leq 1$ is the under-relaxation factor
- When problem reaches steady-state result, $\phi_{P}^{n} = \phi_{P}^{0}$

Rearranging:

$$
\frac{a_P}{\alpha}\phi_{P}^{n}+\sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P} + \bigg(\frac{1-\alpha}{\alpha}a_P\phi_{P}^{0}\bigg)
$$

Since $\alpha$ is less than 1, the diagonal element $\alpha_P$ coefficient is amplified, i.e., diagonal dominance is increased.

### 8.3. Algebraic System Solution

Algorithms like CG method can be used for algebraic solution, which we won't expand on here.

## 9. Summary

In the above discussion, we discussed the basic ideas of the finite volume method, including basic discretization of the four major terms. I believe readers can trace the complete numerical process from continuous governing equations to discrete equations to solving linear algebraic systems. More complex situations are not explored deeply for now.

>[!warning]
>If readers are confused about why discrete equations can be solved as linear algebraic systems, they should briefly review "Numerical Computation" (or similar textbooks like "Scientific Computation") to re-understand the basic ideas of numerical methods. Suggest only appropriate review, not investing大量 time relearning; consult when encountering problems later.

Additionally, because `linear` is repeatedly mentioned, some readers might be confused: isn't this redundant specification?

Actually not, can be summarized as follows:

| Position | Target | Specific Meaning |
|----------|--------|------------------|
| `interpolationSchemes` | All face physical quantities | Global default linear interpolation |
| `gradSchemes` | Face values in gradient computation | Linear interpolation for gradient-specific terms |
| `divSchemes` | Face values in convective term | Linear interpolation for convective term-specific terms |
| `laplacianSchemes` | Face values in diffusive term | Linear interpolation for diffusive term-specific terms |


This article completes discussion of:

- [x] Fundamentals of Finite Volume Method
- [x] Understanding discretization of the four major terms
- [x] Understanding partial correspondence between discrete equations and OpenFOAM code

## References

[1] The Finite Volume Method in Computational Fluid Dynamics, https://link.springer.com/book/10.1007/978-3-319-16874-6

[2] Computational fluid dynamics : the basics with applications, https://searchworks.stanford.edu/view/2989631

[3] Mathematics, Numerics, Derivations and OpenFOAM®, https://holzmann-cfd.com/community/publications/mathematics-numerics-derivations-and-openfoam-free

[4] Notes on Computational Fluid Dynamics: General Principles, https://doc.cfd.direct/notes/cfd-general-principles/

## Support us

>[!tip]
>Hopefully, the sharing here can be helpful to you.
>
>If you find this content helpful, your comments or donations would be greatly appreciated. Your support helps ensure the ongoing updates, corrections, refinements, and improvements to this and future series, ultimately benefiting new readers as well.
>
>The information and message provided during donation will be displayed as an acknowledgment of your support.

{{< cards >}}
  {{< card link="/" title="Support" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="AliPay" >}}
{{< /cards >}}

> Copyright @ 2026 Aerosand
>
> - Course (text, images, etc.)：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
> - Code derived from OpenFOAM：[GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)
> - Other code：[MIT License](https://opensource.org/licenses/MIT)
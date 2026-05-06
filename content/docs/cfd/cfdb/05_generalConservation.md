---
uid: 20251006170601
title: 05_generalConservation
date: 2025-10-06
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
weight: 6
math: true
next:
prev:
comments: true
sidebar:
  exclude: false
draft: false
---


## 0. Preface

We have previously completed the derivation and discussion of the continuity equation (mass equation), momentum equation, and energy equation.

This article mainly discusses:

- [ ] Derivation of the general equation
- [ ] Derivation from different perspectives
- [ ] Understanding the physical meaning of mathematical expressions

## 1. General Equation

Through previous discussions, we can see that no matter what physical quantity, there is always the Reynolds Transport Theorem.

Recall the Reynolds Transport Theorem:

$$
\bigg(\frac{dB}{dt}\bigg)_{MV} = \int_V\bigg[\frac{\partial}{\partial t}(\rho b) + \nabla \cdot (\rho U b)\bigg]dV = \int_V\bigg[\frac{D}{D t}(\rho b) + \rho b \nabla \cdot U\bigg]dV
$$

For a system, physical quantity $\phi$ always has:

**Change of physical quantity in material volume A = Flux change through control volume surface B + Generation/Disappearance within control volume C**


> change of $\phi$ over time $\Delta t$ within the material volume (MV) - A
>
> $=$
>
> surface flux of $\phi$ over time $\Delta t$ across the control volume - B
>
> $+$
>
> source/sink of $\phi$ over time $\Delta t$ within control volume - C


For the first term A:

Combining with the Reynolds Transport Theorem:

$$
A = \frac{d}{dt}\bigg(\int_{MV}(\rho\phi )dV\bigg) = \int_{V}\bigg[\frac{\partial}{\partial t}(\rho\phi) + \nabla\cdot(\rho U\phi)\bigg]dV
$$

Based on previous discussions of velocity divergence and the Reynolds theorem, we can understand that $\rho U \phi$ essentially represents the transport of physical quantity $\phi$ in the flow, called **convective flux**.

We define **convective flux**:

$$F^{C} = \rho U \phi$$

For the second term B:

The second term is the change in physical quantity $\phi$ due to physical phenomena on the control volume surface.

We additionally define **diffusion flux**:

$$F^{D} = -\Gamma^{\phi}\nabla \phi$$

Thus (using the divergence theorem for rearrangement):

$$B = -\int_{\partial V} F^{D}\cdot \vec ndS = -\int_{V} \nabla\cdot F^{D}dV = \int_V\nabla\cdot(\Gamma^{\phi}\nabla\phi)dV$$

> [!tip]
> Flux inward is positive, flux outward is negative. The negative sign is also because diffusion is essentially a "loss" to the system.

For the third term C:

The third term is the change in physical quantity $\phi$ due to physical phenomena within the control volume.

$$C = \int_V S_{\phi}dV$$

Combining all:

$$\int_{V} \bigg[\frac{\partial}{\partial t}(\rho\phi) + \nabla\cdot(\rho\phi U)\bigg]dV = \int_V\nabla\cdot(\Gamma^{\phi}\nabla\phi)dV  + \int_V S_{\phi}dV$$

Let's pause and consider the physical essence of the transport process.

For the fluid we study, due to physical phenomena of convection and diffusion, there should always be the following conservation relationship:

> Increase of physical quantity in the control volume (positive sign)
> $=$
> Inflow of physical quantity through control volume boundaries due to convection mechanism (opposite to control surface direction, negative sign)
> $+$
> Outflow of physical quantity through control volume boundaries due to diffusion mechanism (same as control surface direction, positive sign)
> $+$
> Change of physical quantity due to generation sources (positive sign)

The general conservation equation is summarized as:

$$\frac{\partial}{\partial t}(\rho\phi) + \nabla\cdot(\rho\phi U) - \nabla\cdot(\Gamma^{\phi}\nabla\phi) = S_{\phi}$$

The four major terms in this expression correspond to the four major physical aspects: time term (local time change), convection term (convective change), diffusion term (surface-based diffusion change), and source term (volume-based change).

Some readers may be confused by the complex discussions of diffusion terms, viscous terms, etc., so some supplementary discussion is needed here.

The diffusion term mentioned above is a generalized diffusion term, essentially including mass diffusion in the mass equation, momentum diffusion in the momentum equation, and energy diffusion in the energy equation.

Previously, when discussing the mass equation, we did not discuss material diffusion. Material diffusion or component diffusion generally leans toward chemical engineering, chemistry, and materials science, and will be specifically discussed when encountered later.

Comparing the previously derived N-S equations separately:

1. Mass equation

$$\frac{\partial}{\partial t}\rho + \nabla\cdot(\rho U) = 0$$

2. Momentum equation

$$\frac{\partial}{\partial t}(\rho U) + \nabla \cdot (\rho UU) = \nabla\cdot(\mu\nabla U)-\nabla p + Q$$

3. Energy equation

$$\frac{\partial}{\partial t}(\rho c_pT) + \nabla\cdot(\rho c_p U T) = \nabla\cdot(k\nabla T) + Q^T$$

Summarizing, we have the general form basic equation:

$$\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) = \nabla\cdot(\Gamma\nabla\phi) + S_{\phi}$$

From left to right, we call them: time term (transient term), convection term, diffusion term, source term.

## 2. Summary

So far, the basic equations of fluid dynamics have been largely discussed.

I hope readers can quickly overcome this first hurdle in computational fluid dynamics.

This article completes the discussion of:

- [x] Derivation of the general equation
- [x] Derivation from different perspectives
- [x] Understanding the physical meaning of mathematical expressions


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
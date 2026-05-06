---


uid: 20250918162728
title: 00_cfdbIntro
date: 2025-09-18
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
weight: 1
math: true
next:
prev:
comments: true
sidebar:
  exclude: false
draft: false
---

> [!important]
> Visit [https://aerosand.cc](https://aerosand.cc/) for the latest updates.

## 0. Preface

This series aims to help readers smoothly understand the fundamental theories of computational fluid dynamics.

## 1. Roadmap

{{% steps %}}

### Fluid Dynamics

Theoretical equations

### Finite Difference Method

Basic concepts and methods of finite difference method.

### Finite Volume Method

Basic concepts and methods of finite volume method

{{% /steps %}}

## 2. Mathematical Foundations

### 2.1. Partial Derivative Calculations

Sum and difference rule

$$
\frac{\partial (f+g)}{\partial x} = \frac{\partial f}{\partial x} + \frac{\partial g}{\partial x}
$$

Product rule

$$
\frac{\partial fg}{\partial x} = g\frac{\partial f}{\partial x} + f\frac{\partial g}{\partial x}
$$

Constant

$$
\frac{\partial Cf}{\partial x} = C \frac{\partial f}{\partial x}
$$

### 2.2. Tensor Notation

Tensor notation, also known as Einstein summation convention, is a concise and accurate mathematical expression form.

Repeated indices imply summation.

For example

$$
\frac{\partial \phi_{i}}{\partial x_{i}} = \sum\limits_{i} \frac{\partial \phi_{i}}{\partial x_{i}} = \frac{\partial \phi_{1}}{\partial x_{1}} + \frac{\partial \phi_{2}}{\partial x_{2}} + \frac{\partial \phi_{3}}{\partial x_{3}},i=1,2,3
$$

Different indices are independent.

For example

$$
\begin{align*}
\frac{\partial u_{i}u_{j}}{\partial x_{i}} &= \sum\limits_{i} \frac{\partial u_{i}u_{j}}{\partial x_{i}} \\
&= \begin{pmatrix}
\frac{\partial u_{x}u_{x}}{\partial x} + \frac{\partial u_{y}u_{x}}{\partial y} + \frac{\partial u_{z}u_{x}}{\partial z}\\
\frac{\partial u_{x}u_{y}}{\partial x} + \frac{\partial u_{y}u_{y}}{\partial y} + \frac{\partial u_{z}u_{y}}{\partial z}\\
\frac{\partial u_{x}u_{z}}{\partial x} + \frac{\partial u_{y}u_{z}}{\partial y} + \frac{\partial u_{z}u_{z}}{\partial z}
\end{pmatrix}
\end{align*}
$$

### 2.3. Tensor Operations

Vector inner product

$$
\mathbf{a}\cdot \mathbf{b} =a_{i}b_{i} = \mathbf{a}^{T}\mathbf{b} = a_{1}b_{1} + a_{2}b_{2} + a_{3}b_{3}
$$

Vector and tensor inner product

$$
\mathbf{a}\cdot \mathbf{T} = a_{i}T_{ij} = \begin{pmatrix}a_{1}T_{11}+a_{2}T_{21}+a_{3}T_{31} \\ a_{1}T_{12}+a_{2}T_{22}+a_{3}T_{32} \\ a_{1}T_{13}+a_{2}T_{23}+a_{3}T_{33}\end{pmatrix}
$$

Tensor and vector inner product

$$
\mathbf{T}\cdot\mathbf{a} = T_{ij}a_{j} = \begin{pmatrix}T_{11}a_{1}+T_{12}a_{2}+T_{13}a_{3} \\ T_{21}a_{1}+T_{22}a_{2}+T_{23}a_{3} \\ T_{31}a_{1}+T_{32}a_{2}+T_{33}a_{3}\end{pmatrix}
$$

Tensor double inner product (dyadic product)

$$
\mathbf{T}:\mathbf{S} = T_{ij}S_{ij} = T_{11}S_{11}+T_{12}S_{12}+T_{13}S_{13}+T_{21}S_{21}+T_{22}S_{22}+T_{23}S_{23}+T_{31}S_{31}+T_{32}S_{32}+T_{33}S_{33}
$$

Vector outer product

$$
\mathbf{a} \otimes \mathbf{b} = a_{i}b_{j} = \begin{pmatrix}a_{1}b_{1}&a_{1}b_{2}&a_{1}b_{3} \\ a_{2}b_{1}&a_{2}b_{2}&a_{2}b_{3} \\ a_{3}b_{1}&a_{3}b_{2}&a_{3}b_{3}\end{pmatrix}
$$

### 2.4. Gradient Calculations

Gradient calculation is a dimension-increasing operation.

Scalar gradient

$$
\mathsf{grad}\phi = \nabla\phi = \frac{\partial \phi}{\partial x_{i}} = \begin{pmatrix} \frac{\partial \phi}{\partial x} \\ \frac{\partial \phi}{\partial y}  \\ \frac{\partial \phi}{\partial z} \end{pmatrix}
$$

Vector gradient

$$
\mathsf{grad} \mathbf{U} = \nabla \mathbf{U} = \nabla\otimes \mathbf{U} = \frac{\partial b_{i}}{\partial x_{j}} = \begin{pmatrix} \frac{\partial u_{1}}{\partial x_{1}} & \frac{\partial u_{2}}{\partial x_{1}} & \frac{\partial u_{3}}{\partial x_{1}}  \\ \frac{\partial u_{1}}{\partial x_{2}} & \frac{\partial u_{2}}{\partial x_{2}} & \frac{\partial u_{3}}{\partial x_{2}}  \\ \frac{\partial u_{1}}{\partial x_{3}} & \frac{\partial u_{2}}{\partial x_{3}} & \frac{\partial u_{3}}{\partial x_{3}} \end{pmatrix}
$$

### 2.5. Divergence Calculations

Divergence calculation is a dimension-reducing operation.

Vector divergence

$$
div \mathbf{U} = \nabla\cdot \mathbf{U} = \frac{\partial u_{i}}{\partial x_{i}} = \frac{\partial u_{1}}{\partial x_{1}} + \frac{\partial u_{2}}{\partial x_{2}} + \frac{\partial u_{3}}{\partial x_{3}}
$$

Tensor divergence

$$
div \mathbf{T} = \nabla\cdot \mathbf{T} = \frac{\partial T_{ij}}{\partial x_{i}} = \begin{pmatrix} \frac{\partial T_{11}}{\partial x_{1}}+\frac{\partial T_{21}}{\partial x_{2}}+\frac{\partial T_{31}}{\partial x_{3}} \\ \frac{\partial T_{12}}{\partial x_{1}}+\frac{\partial T_{22}}{\partial x_{2}}+\frac{\partial T_{32}}{\partial x_{3}} \\ \frac{\partial T_{13}}{\partial x_{1}}+\frac{\partial T_{23}}{\partial x_{2}}+\frac{\partial T_{33}}{\partial x_{3}} \end{pmatrix}
$$

### 2.6. Trace of Gradient

Reference gradient calculation

$$
\mathsf{grad} \mathbf{U} = \nabla \mathbf{U} = \nabla\otimes \mathbf{U} = \frac{\partial b_{i}}{\partial x_{j}} = \begin{pmatrix} \frac{\partial u_{1}}{\partial x_{1}} & \frac{\partial u_{2}}{\partial x_{1}} & \frac{\partial u_{3}}{\partial x_{1}}  \\ \frac{\partial u_{1}}{\partial x_{2}} & \frac{\partial u_{2}}{\partial x_{2}} & \frac{\partial u_{3}}{\partial x_{2}}  \\ \frac{\partial u_{1}}{\partial x_{3}} & \frac{\partial u_{2}}{\partial x_{3}} & \frac{\partial u_{3}}{\partial x_{3}} \end{pmatrix}
$$

The trace of this gradient calculation is

$$
\begin{align*}
\mathrm{tr}(\nabla\mathbf{U}) &=  \sum\limits_{i=1}^{3}(\nabla\mathbf{U})_{ii} \\
&=  \frac{\partial u_{1}}{\partial x_{1}} + \frac{\partial u_{2}}{\partial x_{2}} + \frac{\partial u_{3}}{\partial x_{3}}
\end{align*}
$$

We can see that

$$
\nabla\cdot\mathbf{U} = \mathrm{tr}(\nabla\mathbf{U}) = \mathrm{tr}((\nabla\mathbf{U})^{T})
$$

### 2.6. Mixed Calculations

$$
\nabla\cdot(\mathbf{U}\rho) = \mathbf{U}\cdot\nabla\rho + \rho\nabla\cdot \mathbf{U}
$$

$$
\nabla\cdot(\mathbf{U}\otimes \mathbf{U}) = \mathbf{U}\cdot\nabla\otimes \mathbf{U} + \mathbf{U}\nabla\cdot \mathbf{U}
$$

$$
\nabla\cdot(\mathbf{T}\cdot \mathbf{U}) = \mathbf{T}:\nabla\otimes \mathbf{U} + \mathbf{U}\cdot\nabla\cdot \mathbf{T}
$$

### 2.7. Matrix Decomposition

Any matrix can be decomposed into hydrostatic (mean) and deviatoric (non-mean) parts.

$$
\mathbf{A} = \mathbf{A}^{hyd} + \mathbf{A}^{dev}
$$

The magnitude of the hydrostatic part is the sum of diagonal elements, which is also the trace calculation

$$
|\mathbf{A}^{hyd}| = \frac{1}{3}tr(\mathbf{A}) = \frac{1}{3}a_{ii}
$$

The hydrostatic matrix is

$$
\mathbf{A}^{hyd} = \frac{1}{3}tr(\mathbf{A})\mathbf{I}
$$

We can also obtain

$$
\mathbf{A}^{dev} = \mathbf{A} - \mathbf{A}^{hyd} = \mathbf{A} - \frac{1}{3}tr(\mathbf{A})\mathbf{I}
$$

To aid understanding, let's take a 3D matrix as an example

$$
\mathbf{A} = \begin{pmatrix} 1&2&3 \\ 3&2&0 \\ 2&0&3 \end{pmatrix}
$$

The trace of this matrix is

$$
tr(\mathbf{A}) = 1 + 2 + 3 = 6
$$

The hydrostatic part of this matrix is

$$\mathbf{A}^{hyd} = \frac{1}{3}tr(\mathbf{A})\mathbf{I} = 2\mathbf{I} = \begin{pmatrix}2&0&0 \\ 0&2&0 \\ 0&0&2\end{pmatrix}$$

The deviatoric part of this matrix is

$$
\mathbf{A}^{dev} = \begin{pmatrix}-1&2&3 \\ 3&0&0 \\ 2&0&1\end{pmatrix}
$$

### 2.8. Gauss Theorem

Gauss theorem is also known as the divergence theorem

The flux integral over the volume boundary equals the divergence integral within the volume, i.e.

$$
\int_{\partial{V}} \mathbf{U}\cdot d\mathbf{S} = \int_{V}\nabla\cdot \mathbf{U} dV
$$

## 3. Recommendations

It is strongly recommended that readers personally derive the theoretical formulas.

Additionally, be aware that theoretical learning and understanding is an iterative process. It is very normal to consult many books for one knowledge point, or to read one book many times.

When encountering difficulties in learning and understanding, it is recommended not to retreat, but to consult widely, actively discuss, and finally form your own output.

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
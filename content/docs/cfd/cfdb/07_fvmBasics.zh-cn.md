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


## 0. 前言

有限体积法（FVM）基于通量守恒，使得每个控制体都满足物理量守恒（在后续讨论中会逐渐了解），非常适合流体和传热等相关问题的数值计算。FVM 的构造相对简单，精度一般满足要求，推荐 CFD 入门的学习。

我们有流体动力学基本方程的通用形式如下

$$
\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) = \nabla\cdot(\Gamma\nabla\phi) + S_{\phi}
$$

在某一个时间步上，把基本方程对空间离散的单元（网格单元）上做体积积分，有

$$
\int_{V_P}\bigg(\frac{\partial}{\partial t}(\rho \phi) + \nabla \cdot (\rho U\phi) - \nabla\cdot(\Gamma\nabla\phi) \bigg)dV =\int_{V_P} S_{\phi}dV
$$

整理为

$$
\begin{aligned} 
\int_{V_{P}}\bigg(\frac{\partial}{\partial t}(\rho \phi)\bigg)dV + \int_{V_{P}}\bigg(\nabla \cdot (\rho U\phi)\bigg)dV - \int_{V_{P}}\bigg(\nabla\cdot(\Gamma\nabla\phi)\bigg)dV = \int_{V_{P}} S_{\phi}dV \end{aligned}
$$


从左至右依次为时间项、对流项、扩散项、源项。

下面将重点讨论四大项的离散，把握有限体积法的主要思想。至于网格、边界条件、高级离散格式等内容暂不深究。

本文主要讨论

- [ ] 有限体积法基础
- [ ] 了解四大项的离散
- [ ] 了解离散方程与 OpenFOAM 代码的部分对应


## 1. 讨论前

### 1.1. 计算域离散

计算域离散包含着空间离散和时间离散两部分。

从时间变量的角度来说，控制方程是抛物型的方程，所以从初始时间开始，按时间推进计算即可，直到满足计算要求。

从空间变量的角度来说，控制方程可能是双曲或者圆锥型的，需要使用不同的算法解决。

>[!tip]
>关于抛物型、双曲型、圆锥形方程的区别，暂不深究，在学习中会逐渐理解。

有限体积法把整个空间计算域划分成彼此不重叠又连续的控制体。

### 1.2. 约定说明

为了书写方便，约定除非特别说明，大写字母根据物理意义优先表示物理量的矢量

- 速度矢量 $U=(U_{x},U_{y},U_{z})$
- 压力标量 $p$
- 任一待求物理量 $\phi$
- 扩散系数 $\Gamma^{\phi}$，和物理量 $\phi$ 有关，如非特别说明默认简单写成 $\Gamma$
- 源项 $Q^{\phi}$ ，和物理量 $\phi$ 有关，如非特别说明默认简单写成 $Q$
- 面矢量 $S=(S_{x},S_{y},S_{z})$，方向始终向外

上下标符号有

- 上标 $t$ 表示当前时间步（已知量），等同于上标 $o$，即旧时间步（old）
- 上标 $t+1$ 表示新时间步（待求量），等同于上标 $n$ ，即新时间步（new）
- 上标 $t+n$ 表示依次类推的之后时间步
- 上标 $*$ 表示基于算法迭代的中间预测值
- 下标 $P$ 表示当前单元体心（取 Owner 的 O 实在很容易混淆）
- 下标 $N$ 表示相邻单元体心（Neighbor）
- 下标 $f$ 表示当前单元和相邻单元之间的界面的面心（face），也模糊表示单元面

网格符号

- 体积 $V$，如单元的总体积 $V_P$  
- 面积 $\partial{V}$ ，如单元的表面积 $\partial{V_{P}}$
- 单元界面矢量 $S_{f}$，其面积大小为 $|S_{f}|$

对于内部面，面矢量从序号小控制体（owner）的指向序号大的控制体（neighbor），对于边界面，面矢量始终指向外。

对于某个控制体来说，当遍历这个控制体的所有控制面时

- 如果邻单元序号比本单元序号大，那么两者之间的面矢量，就是从本单元指向邻单元
	- 对于本控制体来说，该面矢量向外，与积分和通量的数学约定相同，所以为正
	- 对于该面来说，序号小的单元序号就被存在 owner 数组中，序号大的存在 neighbor 数组中
- 如果邻单元序号比本单元序号小，那么两者之间的面矢量 ，就是从邻单元指向本单元
	- 对于本控制体来说，该面矢量向内，与积分和通量的数学约定相反，所以为负
	- 对于该面来说，序号小的单元序号仍被存在 owner 数组中，序号大的存在 neighbor 数组中
- 对于面的 owner 数组来说，自然序号就是面的序号，元素的值就是该序号面的 owner 单元的序号， neighbor 数组类似
- 当计算某个单元，遍历到某个面时，如果这个面经过索引，发现该单元是该面的 owner，那么在该单元的计算中，该面的面矢量就是正的，如果发现该单元是该面的 neighbor，那么在该单元的计算中，该面的面矢量就是负的

>[!tip]
>后续讨论到序号的时候，可以再回顾这里对序号的讨论

### 1.3. 控制体

控制体是有限体积法中将连续计算域划分出来的小体积单元，一般也就是网格划分中的网格单元。

我们通过体积来定义中心，中心点矢量 $\vec{x}_{P}$ 乘以体积 $V_{P}$，等于该空间内各个点的位置矢量 $\vec{x}$ 在体积上的积分，即有

$$
V_{P}\vec x_{P} = \int_{V_{P}}\vec{x} dV
$$

将体积替换为体积分，移项后整理为

$$
\int_{V_{P}}\vec{x}dV - \vec{x}_P\int_{V_{P}}dV = 0
$$

继续整理，有

$$
\int_{V_{P}}(\vec{x} - \vec{x}_{P})dV= 0
$$

上式即为一个空间体积的中心点 $\vec{x}_{P}$的定义。

### 1.4. 离散精度

对于其中的连续场 $\phi$ 来说，进行离散后的场总是可以使用泰勒展开表示

空间上

$$
\phi(\vec{x}) \approx \phi_{P}+ (\vec{x} - \vec{x}_{P})\cdot(\nabla\phi)_{P}
$$

时间上

$$
\phi^{t+\Delta t} \approx \phi^{t} + \Delta t(\frac{\partial \phi}{\partial t})^{t}
$$

上面两种离散格式具有二阶精度，基本满足计算流体力学的偏微分方程求解。


### 1.5. 函数积分

基于泰勒展开二阶近似和控制体中心点的定义，连续函数的处理如下

$$
\begin{aligned}
\int_{V_{P}}\phi(\vec{x})dV &\approx \int_{V_{P}}\bigg[\phi_{P}+ (\vec{x} - \vec{x}_{P})\cdot(\nabla\phi)_{P}\bigg]dV \\
&= \phi_{P}\int_{V_{P}}dV + \bigg[\int_{V_{P}}(\vec{x} - \vec{x}_{P})dV\bigg]\cdot(\nabla\phi)_{P} \\
&= \phi_{P}V_{P}
\end{aligned}
$$

整理有

$$
\int_{V_{P}}\phi(\vec{x})dV = \phi_{P}V_{P}
$$

即某个连续函数在某个体积空间的积分，近似等于，该函数在该体积空间的中心点值直接乘以体积。

### 1.6. 散度积分

我们知道高斯定理有

对于向量有

$$
\int_{V}\nabla\cdot\vec{a}dV = \int_{\partial V}\vec{a}\cdot d\vec{S}
$$

对于标量有

$$
\int_V\nabla\phi dV = \int_{\partial V}\phi d\vec{S}
$$

应用高斯定理，有体积分转面积分（无近似）

矢量场散度计算的体积积分，使用高斯定理有

$$
\begin{aligned}
\int_{V_{P}}\nabla\cdot\vec{a}dV = \int_{\partial V_{P}}\vec{a}\cdot d\vec{S} 
\end{aligned}
$$

物理量的散度计算的体积积分，本质上体积边界上的通量积分等于体积内的散度积分。

面积分离散（无近似）

$$
\int_{\partial V_{P}}\vec{a}\cdot d\vec{S} = \sum\limits_{f}(\int_f\vec{a}\cdot d\vec{S})
$$

进一步离散为，各个离散面上物理量和面矢量计算的面积分的求和。

数值积分近似（近似）

积分的数值求积本质上是用求和代替积分，也被称为机械求积。被积函数在多个离散点求值，然后求和。

具体到此处的计算来说，

$$
\int_{f}\vec{a}\cdot d\vec{S} \approx  \sum\limits_{ip}\omega_{ip}(\vec{a}_{f}\cdot \vec{S}_f)_{ip}
$$

其中 $\omega_{ip}$ 为积分节点。该近似的本质是，连续的积分计算近似于积分节点代数计算的累积加权。当单元足够小，单元的离散面也足够小，取离散面的中心点作为唯一的积分节点也可以满足精度要求。

此时，上式也就近似为

$$
\int_{f}\vec{a}\cdot d\vec{S} \approx  \vec{a}_{f}\cdot \vec{S}_{f}
$$

代回散度计算，有（虽然是近似，以后方便起见直接写等号）

$$
\int_{V_{P}}\nabla\cdot\vec{a}dV = \sum\limits_{f}\vec{a}_{f}\cdot \vec{S}_{f}
$$

在足够小的控制体上，散度积分和体积无关，可以当作独立连续函数，连续函数可以按上文方式处理，最后有

$$
(\nabla\cdot\vec{a})V_{P}= \sum\limits_{f}\vec{a}_{f}\cdot \vec{S}_{f}
$$

再次注意，我们约定，只有本单元对于该面来说是 owner 的时候， $S_{f}$ 才从 点 P 向外指向点 N 为正，当本单元对于该面来说是 neighbor 的情况下，$S_{f}$ 从点 N 向内指向点 P ，与数学定义相反，为负，所以上式其实可以写为

$$
\sum\limits_{f}\vec{a}_{f}\cdot \vec{S}_{f}= \sum\limits_{owner}\vec{a}_{f}\cdot \vec{S}_{f} - \sum\limits_{neighbor}\vec{a}_{f}\cdot \vec{S}_{f}
$$


## 2. 对流项

对流项的空间积分为

$$
\int_{V_{P}}\nabla\cdot(\rho U\phi)dV
$$

### 2.1. 对流离散

对流项离散过程为

$$
\begin{aligned}
\int_{V_{P}}\nabla\cdot(\rho U\phi)dV &= \int_{\partial V_{P}}(\rho U\phi)\cdot d S \\
&= \sum\limits_{f}\int_{f}(\rho U\phi) \cdot d S\\
&\approx \sum\limits_{f}(\rho U\phi)_{f}\cdot S_{f} =  \sum\limits_{f}F^{C}_{f}\cdot S_{f} \\
&= \sum\limits_{f}(\rho U_{f}\cdot S_{f})\phi_{f} =  \sum\limits_{f}F_{f}\phi_{f}
\end{aligned}
$$

>[!tip]
>讨论刚刚开始，为了让阅读更加流畅和清晰，有些内容会反复提及，以后会适当跳过。

离散的计算过程具体分析如下

体积分转面积分（无近似）

$$
\int_{V_{P}}\nabla\cdot(\rho U\phi)dV = \int_{\partial V_{P}}(\rho U\phi)\cdot dS
$$

面积分离散（无近似）

$$
\int_{\partial V_{P}}(\rho U\phi)\cdot d S = \sum\limits_{f}\int_{f}(\rho U\phi) \cdot dS
$$

- 单元表面是一个完整的闭合面，由若干离散面组成，例如六面体网格由六个离散面组成
- 面积分等于这六个离散面积分的和（没有近似）

对于单个离散面上的面积分来说，数值求积有（近似）

$$
\int_{f}(\rho U\phi)_{f} \cdot dS \approx \sum\limits_{ip}\omega_{ip}(\rho U\phi)_{ip}\cdot S_{ip}
$$

对于数值计算网格来说，当网格很小的时候，一点近似足够，即

$$
\int _{f}(\rho U\phi)_{f}\cdot dS \approx (\rho U\phi)_{f}\cdot S_{f} = \sum\limits_{f}(\rho U_{f}\cdot S_{f})\phi_{f}
$$

需要注意的是，这一点取在离散面的中心上。

结合上面的讨论，扩散项的半离散近似最终为

$$
\int_{V_{P}}\nabla\cdot (\rho U\phi)dV  \approx \sum\limits_{f}(\rho U\phi)_{f}\cdot S_{f} =  \sum\limits_{f}F^{C}_{f}\cdot S_{f} = \sum\limits_{f}F_{f}\phi_{f}
$$

其中，$F^{C}$ 称为对流通量（矢量），反映了对流的物理本质。

$$
F^{C}_{f} = (\rho U\phi)_{f}
$$

如果其中的 $\phi=U$ ，则

$$
F^{C}_{f} = (\rho U U)_{f}
$$

这里 $F^{C}$ 也称为动量通量。

而 $F_{f}$ 称为质量通量（标量），

$$
F_{f} = \rho U_{f}\cdot S_{f}
$$

质量通量反映了对流过程中更基本的物理守恒本质。

>[!tip]
>有人可能对点乘的面矢量的位置有疑问。实际上，向量点乘满足交换律，所以位置在前后都可以。


总的来说，对流项就是物理量随着质量输运的变化，数学上具体表现为截面上物理量乘以截面上的质量通量。

### 2.2. 对流插值

注意到对流项离散后中的各项值是面中心的值，如 $(\rho U)_{f}$。对于有限体积法来说，各个值都是控制体中心的值，所以我们需要从体中心值插值得到面中心值。

我们可以使用与面最近的两个体中心值来插值。

**中央差分（Central Differencing / CD）**

$$
\phi_{f}= f_{x}\phi_{P}+ (1-f_{x})\phi_{N}
$$

插值因子为

$$
f_{x}= \frac{|fN|}{|PN|} 
$$

- 文献已经证明CD格式是二阶精度的，即使非均匀网格也是
- 在对流主导的流动中可能产生非物理的振荡
- 可能违背解的有界性

>[!tip]
>暂不理解没有关系，随着讨论推进，困惑会逐渐减少。

**迎风差分 （Upwind Differencing / UD）**

$$
\phi_{f}= f_{x}\phi_{P}+ (1-f_{x})\phi_{N}
$$

其中的系数为

$$
f_{x} =  
\begin{cases}  
1, \phi_{f} =  \phi_{P},  \space for F \geq  0 \\  
0, \phi_{f} =  \phi_{N},  \space for F  \leq 0  
\end{cases}
$$

也就是面中心值始终取上游的体中心值。

- 文献已经证明 UD 是无条件的有界且稳定
- 不会产生非物理振荡
- 只有一阶精度，求解可能偏差很大

**混合差分 （Blended Differencing / BD）**

为了同时保证精度和有界性，可以构造 CD 格式和 UD 格式的线性组合

$$
\phi_{f}= (1-\gamma)(\phi_{f})_{UD} + \gamma(\phi_{f})_{CD}
$$

其中 $(\phi_{f})_{UD}$ 是基于 UD 格式的值，$(\phi_{f})_{CD}$ 是基于 CD 格式的值。$\gamma$ 是混合因子，取值为 $0 \leq \gamma\leq 1$。

或者写成一个整体

$$
\begin{aligned}
\phi_{f} &= \bigg[(1-\gamma)\max(sgn(F_{f}),0) + \gamma f_{x}\bigg]\phi_{P} \\
&+ \bigg[(1-\gamma)\min(sgn(F_{f}),0) + \gamma(1-f_{x})\bigg]\phi_{N}
\end{aligned}
$$

- 文献证明 BD 格式可以有效同时保证精度和有界性

OpenFOAM 中提供对应的 `Gauss upwind` 格式。OpenFOAM 的 `fvSchemes` 中可以指定

```cpp
divSchemes
{
	default Gauss upwind;
}
```

其中的，`Gauss` 表示离散是基于高斯定理，`upwind` 表示插值方式采用 upwind 格式。OpenFOAM 还提供其他可选项，暂不深究。


## 3. 扩散项

扩散项的体积积分为

$$
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV
$$

### 3.1. 扩散离散

$$
\begin{align*}
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV &= -\int_{\partial V_{P}}(\Gamma\nabla \phi)\cdot dS\\
&= -\sum\limits_{f}\int_{f}(\Gamma\nabla \phi)\cdot dS\\
&\approx -\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f}\\
&= \sum\limits_{f}F^{D}_{f}\cdot S_{f}
\end{align*}
$$

离散的计算过程具体分析如下

体积分转面积分（无近似）

$$
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV = -\int_{\partial V_{P}}(\Gamma\nabla \phi)\cdot dS
$$

面积分离散（无近似）

$$
-\int_{\partial V_{P}}(\Gamma\nabla \phi)\cdot dS = -\sum\limits_{f}\int_{f}(\Gamma\nabla \phi)\cdot dS
$$

- 单元表面是一个完整的闭合面，由若干离散面组成，例如六面体网格由六个离散面组成。
- 面积分可以等于这六个离散面积分的和（没有近似）

数值积分（近似）

对于单个离散面上的面积分来说，数值求积有

$$
-\int_{f}(\rho\Gamma\nabla \phi)\cdot dS \approx -\sum\limits_{ip}\omega_{ip}(\rho\Gamma_{\phi}\nabla \phi)_{ip}\cdot S_{ip}
$$

对于数值计算网格来说，当网格很小的时候，一点近似足够，即

$$
-\int_{f}(\Gamma\nabla \phi)\cdot dS \approx -(\rho\Gamma\nabla \phi)_{f}\cdot S_{f}
$$

需要注意的是，这一点取在离散面的中心上。

结合上面的讨论，扩散项的半离散近似最终为

$$
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV \approx -\sum_{f}(\Gamma\nabla \phi)_f\cdot S_{f} = \sum\limits_{f}F^{D}_{f}\cdot S_{f}
$$

其中，$F^{D}$ 称为扩散通量

$$
F^{D}_{f} = (-\Gamma\nabla\phi)_{f}
$$

>[!tip]
>关于负号的理解：
>1. 对于通用方程来说，除了源项，时间项、对流项、扩散项都置于左侧，在数学上组建为通量的累加。
>2. 物理上通量方向与梯度方向相反

### 3.2. 梯度插值

对于正交网格来说，梯度可以按面矢量方向 $\vec{n}$ 与面矢量垂直方向 $\vec{\tau}$ 进行计算

$$(\nabla \phi)_{f} = (\frac{\partial \phi}{\partial \vec{n}})_{f} + (\frac{\partial \phi}{\partial \vec{\tau}})_{f}$$

和面矢量点乘，垂直方向分量为零，只余下法向方向分量

$$
(\nabla \phi)_{f}\cdot S_{f} = (\frac{\partial \phi}{\partial \vec{n}})_{f}|S_{f}|
$$

余下部分的梯度计算按距离插值，有

$$
(\frac{\partial \phi}{\partial \vec{n}})_{f} = \frac{\phi_{N} - \phi_{P}}{|\vec{d}|}
$$

即

$$
(\nabla \phi)_{f} \cdot S_{f} =\frac{\phi_{N} - \phi_{P}}{|\vec{d}|} |S_{f}|
$$

也可以对梯度计算按线性插值，有

$$
(\nabla \phi)_{f}= f_x(\nabla \phi)_{P}+(1-f_x)(\nabla \phi)_N
$$

其中的插值系数可以参考前文的 CD 格式的系数。

这些方式均将面中心的梯度计算转换成了体中心的梯度计算的线性插值。

### 3.3. 梯度计算

如果按照上面的线性插值之后，我们仍然要计算体中心的梯度

$$
(\nabla \phi)_{P}
$$

若作体积分，利用高斯定理，利用面积分离散，利用数值积分，有

$$
\int_{V_{P}}(\nabla \phi)_{P}dV = \int_{\partial V_{P}}\phi dS = \sum\limits_{f}\int_{f}\phi dS \approx \sum\limits_{f}\phi_{f}S_{f}
$$

而体积分又等于

$$
\int_{V_{P}}(\nabla \phi)_{P}dV = V_{P}(\nabla \phi)_{P}
$$

所以可以得到

$$
(\nabla \phi)_{P} = \frac{1}{V_{P}}\sum\limits_{f}\phi_{f}S_{f}
$$

上式中的面值 $\phi_{f}$ 可以通过线性插值得到

$$
\phi_{f} = (1-\lambda)\phi_{P} + \lambda\phi_{N}
$$

OpenFOAM 中提供对应的 `Gauss linear` 格式。OpenFOAM 的 `fvSchemes` 中可以指定

```cpp
gradSchemes
{
	default Gauss linear;
}
```

其中的，`Gauss` 表示离散是基于高斯定理，`linear` 表示插值方式采用线性插值格式，系数使用单元中心距离比例。OpenFOAM 还提供其他可选项，暂不深究。

### 3.4. 非正交修正

非正交网格的面矢量 $S_{f}$ 可以分解成

$$\vec S_f = \vec \Delta_f + \vec k_f$$

从而

$$
(\nabla \phi)_{f}\cdot S_{f} = (\nabla \phi)_{f}\cdot\vec{\Delta} + (\nabla \phi)_{f}\cdot\vec{k}
$$

面矢量的分解中，我们把 $\vec{\Delta}$ 选择为与 $\vec{d}$ （本单元体中心指向邻单元体中心的方向矢量）平行，这样的话，正交影响项就可以继续使用【正交网格梯度计算】的方法。网格的非正交导致的偏差由非正交修正项来修正。

面矢量的分解可以有以下方法

**最小修正方法**

以 $\vec{S}$ 为斜边，$\vec{\Delta}$ 和 $\vec{k}$ 为直角边构造分解。

此时有

$$
\vec{\Delta} = \frac{\vec{d}\cdot\vec{S}}{\vec{d}\cdot\vec{d}}\vec{d}
$$

相应的 $\vec{k}$ 由面矢量分解出来。

这种方法下，当网格非正交性增加，面矢量向 $\vec{k}$ 分解的更多，也就是 $\phi_P$ 和 $\phi_N$ 的影响减小。

**正交修正方法**

以 $\vec{S}$ 和 $\vec{\Delta}$ 为等腰边，$\vec{k}$ 为第三边构造分解。

此时有

$$
\vec{\Delta} = \frac{\vec{d}}{|\vec{d}|}|\vec{S}|
$$

这种方法下， $\phi_P$ 和 $\phi_N$ 的影响不随着非正交性变化而变化。

**超松弛方法**

以 $\vec{S}$ 和 $\vec{k}$ 为直角边，$\vec{\Delta}$ 为斜边构造分解。

此时有

$$
\vec{\Delta} = \frac{\vec{d}}{\vec{d}\cdot\vec{S}}|\vec{S}|^2
$$

这种方法下，当网格非正交性增加，面矢量向 $\vec{\Delta}$ 分解的更多，也就是 $\phi_P$ 和 $\phi_N$ 的影响增加。

对于 surface normal gradient，OpenFOAM 中提供对应的 `orthogonal` 格式。OpenFOAM 的 `fvSchemes` 中可以指定

```cpp
snGradSchemes
{
	default orthogonal;
}
```

其中的，`orthogonal` 表示假设网格正交，使用最简单的差分格式。OpenFOAM 还提供其他可选项，暂不深究。


### 3.5. 最终形式

对于正交影响的部分来说，如果按距离插值（书写简单起见）

$$
(\nabla \phi)_{f}\cdot\vec{\Delta} = \frac{\phi_{N}- \phi_{P}}{|\vec{d}|}\cdot \vec{\Delta}
$$

最终形式为

$$
(\nabla \phi)_{f}\cdot S_{f} = \frac{\phi_{N}- \phi_{P}}{|\vec{d}|}\cdot \vec{\Delta} + (\nabla \phi)_{f}\cdot\vec{k}
$$

上式 RHS 第二项面上的梯度计算可以参考前文，按体积插值得到。

结合之前的扩散项整理结果

$$
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV \approx -\sum_{f}(\Gamma\nabla \phi)_f\cdot S_{f} = \sum\limits_{f}F^{D}_{f}\cdot S_{f}
$$

代入计算后的扩散项为

$$
\begin{align*}
-\int_{V_{P}}\nabla\cdot (\Gamma \nabla \phi)dV &=  -\int_{\partial V_{P}}(\Gamma\nabla \phi)\cdot dS \\
&\approx -\sum_{f}(\Gamma\nabla \phi)_f\cdot S_{f} \\
&=  -\sum_{f}\Gamma_{f}\bigg[\frac{\phi_{N} - \phi_{P}}{|\vec{d}|}\cdot\vec{\Delta} + (\nabla \phi)_f\cdot\vec k\bigg]
\end{align*}
$$

如果扩散系数是均匀，则可以直接结算。如果扩散系数不是均匀的，则同样需要进行类似的插值近似计算。如

$$
\Gamma_{f} = (1 - \lambda)\Gamma_{P}+ \lambda \Gamma_{N}
$$

OpenFOAM 中提供对应的 `Gauss linear orthogonal` 格式。OpenFOAM 的 `fvSchemes` 中可以指定

```cpp
laplacianSchemes
{
	default Gauss linear orthogonal;
}
```

其中的，`Gauss` 表示离散是基于高斯定理，`linear` 表示梯度面值插值方式采用 linear 格式，即

$$
(\nabla \phi)_{f}= f_x(\nabla \phi)_{P}+(1-f_x)(\nabla \phi)_N
$$

而 `orthogonal` 表示只考虑正交影响，忽略非正交修正。OpenFOAM 还提供其他可选项，暂不深究。

## 4. 源项

广义来说，所有不能归入对流项或扩散项或时间项的都被当成源项来处理。

一般会先考虑源项能否线性化，如果可以线性化

$$
S_{\phi}(\phi) = Su + Sp\phi
$$

空间积分为

$$
\int_{V_{P}}S_{\phi}(\phi)dV = SuV_{P}+ SpV_{P}\phi_{P}
$$

我们特别讨论一下压力源项。如果压力项进行显性离散，压力使用上一步已知压力，有

$$
\begin{align*}
\int_{V_{P}}\nabla pdV &= \int_{\partial V_{P}}pdS \\
&= \sum_{f}p_{f}S_{f} \\
&= \sum_{f}p_{f}^{t}S_{f}
\end{align*}
$$

注意，上面的 $S_{f}$ 按约定是面矢量。

源项在处理的时候，需要注意的是，线性化处理可能导致移项后， LHS的系数矩阵对角占优减弱，从而影响计算的稳定性。所以，源项处理要注意 LHS 系数矩阵的对角占优。

## 5. 时间项



时间项的体积分为，

$$\int_{V_{P}}\bigg(\frac{\partial}{\partial t}(\rho \phi)\bigg)dVdt$$

对时间的偏导和体积没有任何关系，所以上式整理有

$$
\int_{V_{P}}\bigg(\frac{\partial}{\partial t}(\rho \phi)\bigg)dVdt = \frac{\partial}{\partial t}(\rho \phi)\int_{V_{P}}dV = V_{P}\frac{\partial \rho \phi}{\partial t}
$$

对时间的偏导还需要进一步处理。

## 6. 时间离散

综合前文空间离散的讨论，我们进一步讨论时间离散。

对基本方程进行一个时间步的时间积分，有

$$
\begin{aligned} 
\int_{t}^{t+1} \bigg[\int_{V_P}\bigg(\frac{\partial}{\partial t}(\rho \phi)\bigg)dV + \int_{V_P}\bigg(\nabla \cdot (\rho U\phi)\bigg)dV &- \int_{V_P}\bigg(\nabla\cdot(\Gamma\nabla\phi)\bigg)dV \bigg]dt\\
&= \int_{t}^{t+1}\bigg[\int_{V_P} S_{\phi}dV \bigg]dt
\end{aligned}
$$

将前文的讨论代入，假设源项可以线性化有

$$\begin{aligned}
\int_{t}^{t+1}\bigg[\frac{\partial \rho\phi}{\partial t}V_{P}+\sum\limits_{f}F^{C}_{f}\cdot S_{f}&+ \sum\limits_{f}F^{D}_{f}\cdot S_{f} \bigg]dt\\
&= \int_{t}^{t+1}\bigg[SuV_{P}+ SpV_P\phi_{P}\bigg]dt
\end{aligned}
$$

进一步代入有

$$\begin{aligned}
\int_{t}^{t+1}\bigg[\frac{\partial \rho\phi}{\partial t}V_{P}+\sum\limits_{f}(\rho U\phi)_{f}\cdot S_{f}&-\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f} \bigg]dt\\
&= \int_{t}^{t+1}\bigg[SuV_{P}+ SpV_P\phi_{P}\bigg]dt
\end{aligned}
$$

上面这个表达式一般被称为半离散方程。

考虑到连续场在时间上的变化

时间项有

$$
\bigg(\frac{\partial \rho \phi}{\partial t}\bigg)_{P} = \frac{\rho^{t+1}_{P}\phi^{t+1}_{P}-\rho^{t}_{P}\phi^{t}_{P}}{\Delta t}
$$

对于连续函数的时间积分有

$$
\int_{t}^{t+1}\phi(t)dt \approx ( (1-f)\phi^{t+1} + f\phi^{t} )\Delta t 
$$

系数 $f$ 取不同的值对应着不同的离散格式，有

$$
\begin{aligned}  
\frac{\rho^{t+1}_{P}\phi^{t+1}_{P}-\rho^{t}_{P}\phi^{t}_{P}}{\Delta t} V_{P}&+\\
\sum\limits_{f}\bigg[(1-f)F_{f}^{C(t+1)}\cdot S_{f} + fF^{C(t)}_{f}\cdot S_{f} \bigg] &- \sum\limits_{f} \bigg[(1-f)F_{f}^{D(t+1)}\cdot S_{f}+fF_{f}^{D(t)}\cdot S_{f} \bigg]\\
&=  SuV_{P}+(1-f)S_pV_P\phi_{P}^{t+1} + fS_pV_P\phi_{P}^{t}  
\end{aligned}
$$

### 6.1. 欧拉格式

当系数取 $f=0$ 时，这种格式在 OpenFOAM 中称为欧拉格式。

此时如果对流项和扩散项使用 `fvm` 进行隐式离散，离散方程为

$$
\begin{aligned}  
\frac{\rho^{t+1}_{P}\phi^{t+1}_{P}-\rho^{t}_{P}\phi^{t}_{P}}{\Delta t} V_{P} &+\\ \sum\limits_{f}F^{C(t+1)}_{f}\cdot S_{f}&- \sum\limits_{f} F_{f}^{D(t+1)}\cdot S_{f}\\
&=  SuV_{P} + S_{p}V_{P}\phi_{P}^{t+1}  
\end{aligned}
$$

根据前文的讨论，$F_{f}^{C},F_{f}^{D}$ 据可以插值为本单元（下标$P$）和邻单元（下标$N$）的中心值计算组合。

将所有的未知量（新时间步）至于 LHS，已知量（旧时间步）和源项等至于 RHS ，最后可以整理为线性代数系统进行求解

$$
a_{P}\phi_{P}^{n}+\sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P}
$$

这种离散方程在时间上具有一阶精度空间上具有二阶精度，数值计算相对简单，小时间步稳定。

OpenFOAM 中提供对应的 `Euler` 时间格式，在 `fvSchemes` 中指定为

```cpp
ddtSchemes
{
	default    Euler;
}
```


### 6.2. CN 格式

当系数取某个值（如 0.5）时，这种格式在 OpenFOAM 中称为半隐式 Crank-Nicolson 格式。

此时如果对流项和扩散项使用 `fvm` 进行隐式离散，离散方程为

$$
\begin{aligned}  
\frac{\rho^{t+1}_{P}\phi^{t+1}_{P}-\rho^{t}_{P}\phi^{t}_{P}}{\Delta t} V_{P}&+\\
\sum\limits_{f}\bigg[\frac{1}{2}F_{f}^{C(t+1)}\cdot S_{f} + \frac{1}{2}F^{C(t)}_{f}\cdot S_{f} \bigg] &- \sum\limits_{f} \bigg[\frac{1}{2}F_{f}^{D(t+1)}\cdot S_{f}+ \frac{1}{2}F_{f}^{D(t)}\cdot S_{f} \bigg]\\
&=  SuV_{P}+ \frac{1}{2}S_pV_P\phi_{P}^{t+1} + \frac{1}{2}S_pV_P\phi_{P}^{t}  
\end{aligned}
$$

可以看到，CN格式对空间项（对流项、扩散项、源项）也进行了时间加权平均。

同样的，将所有的未知量（新时间步）至于 LHS，已知量（旧时间步）和源项等至于 RHS ，最后可以整理为线性代数系统进行求解

$$
Ax = b
$$

经过数学验证，这种 CN 格式在时间上具有二阶精度空间上具有二阶精度。

OpenFOAM 中提供对应的 `CrankNicolson` 时间格式。OpenFOAM 的 `fvSchemes` 中可以指定

```cpp
ddtSchemes
{
	default    CrankNicolson 0.5;
}
```

### 6.3. backward 格式

这种 backward 格式仍然是全隐式格式，不过提高了时间项的离散精度。

此时如果对流项和扩散项使用 `fvm` 进行隐式离散，离散方程为

$$
\begin{aligned}  
\frac{\frac{3}{2}\rho^{t+1}_{P}\phi^{t+1}_{P}-2\rho^{t}_{P}\phi^{t}_{P}+ \frac{1}{2}\rho^{t-1}_{P}\phi^{t-1}_{P}}{\Delta t} V_{P} &+\\ \sum\limits_{f}F^{C(t+1)}_{f}\cdot S_{f}&- \sum\limits_{f} F_{f}^{D(t+1)}\cdot S_{f}\\
&=  SuV_{P} + S_{p}V_{P}\phi_{P}^{t+1}  
\end{aligned}
$$

可以看到，这种格式的时间项采用了三层时间步，空间项（对流项、扩散项、源项）则完全使用新时间步的值。

同样，离散方程组可以整理为线性代数系统进行求解。

经过数学验证，这种格式在时间上具有二阶精度空间上具有二阶精度。

OpenFOAM 中提供对应的 `backward` 时间格式。OpenFOAM 的 `fvSchemes` 中可以指定

```cpp
ddtSchemes
{
	default    backward;
}
```


## 7. 边界条件

我们分成数值边界（numerical boundary conditions）和物理边界（physical boundary conditions）进行讨论。

### 7.1. 数值边界

数值边界有两种基本类型

- Dirichlet (fixed value of variable to the boundary)（指定边界上变量的值）
- Neumann (fixed gradient of variable normal to the boundary)（指定边界上变量的法向梯度值）

此外可能还会出现这两种基础类型的不同混合形式。

这些类型需要在线性代数系统求解之前，应用到代数系统中。

**非正交性**

在正式将数值边界条件应用到线性代数系统之前，我们需要考察一下网格非正交性带来的影响。

对于一个边界控制体，控制体体中心点还是 $P$ ，边界面中心点是 $b$ ，体中心点到边界面中心点的向量为 $\vec d$ ，因为非正交性，体中心点到边界面的法向向量为 $\vec{d}_{n}$ 。

如果还是对面矢量 $\vec{S}$ 作分解 $\vec{S} = \vec{\Delta} + \vec{k}$ ，因为边界面之外不存在邻单元，我们简单认为 $\vec{\Delta}$ 和 $\vec{S}$  相同。

$$
\vec{d}_{n}= \frac{\vec{S}}{|\vec{S}|}\frac{\vec{d}\cdot{S}}{|\vec{S}|}
$$

此时不再使用非正交修正项 $\vec{k}$ 。

我们也可以继续使用非正交修正项，以提高离散精度。

**Fixed Value BC**

也就是在边界面上

$$
\phi_f = \phi_b
$$
对流项和扩散项都处理该值。

对于对流项来说

$$
\int_{V_{P}}\nabla\cdot(\rho U\phi)dV = \sum\limits_fF_{f}\phi_{f}
$$

所以当面求和遍历到边界面时，应该直接替换为定值

$$
F_{f}\phi_{f} = F_b\phi_b
$$

对于扩散项来说

$$
-\int_{V_{P}}\nabla\cdot(\Gamma \nabla\phi)dV = -\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f}
$$

在计算 $(\nabla \phi)_{f}\cdot S_{f}$  时，也需要使用定值

$$\begin{aligned}
(\nabla \phi)_{f}\cdot S_{f} &= |\vec{\Delta}|\frac{\phi_{N}- \phi_{P}}{|\vec{d}|} + \cancel{\vec{k}\cdot(\nabla\phi)_{f}} \\
&= |\vec{S}|\frac{\phi_{b}- \phi_{P}}{|\vec{d}_{n}|}
\end{aligned}$$

**Fixed Gradient BC**

也就是在边界上

$$
\bigg(\frac{\vec S}{|\vec S|}\cdot \nabla \phi\bigg)_{b} = g_{b}
$$

对于对流项来说

梯度可以反推计算

$$\begin{aligned}
\phi_{b} &= \phi_{P} + \vec d_{n}\cdot (\nabla\phi) \\
&= \phi_{P} + |\vec d_{n}|g_{b}
\end{aligned}$$

得到定值再代入即可。

注意，因为网格的非正交性， $\vec{d}_{n}$ 并不是完全指向边界面中心点，所以 Fixed Gradient BC 的计算是一阶精度的。我们可以增加非正交修正项来提高精度。

对于扩散项来说

面上的梯度直接代入计算

$$
-\int_{V_{P}}\nabla\cdot(\Gamma \nabla\phi)dV = -\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f}
$$

当面求和遍历到边界面时，

$$
\sum_{f}(\Gamma\nabla \phi)_{f}\cdot S_{f}= \Gamma_{b}|S|g_b
$$

### 7.2. 物理边界

**不可压缩流动**

- Inlet BC
	- velocity: fixed value
	- pressure: fixed gradient zero
- Outlet BC
	- 速度可以从相邻单元映射过来，并缩放以满足整体连续性
		- 压力指定为零梯度
		- 可能导致不稳定
	- 速度可以指定为零梯度
		- 压力采用固定值
		- 通过压力方程保证整体质量守恒
- Symmetry plane BC
	- 边界上的法向梯度都为零
- Impermeable no-slip walls
	- 流体在壁面上的速度等于壁面本身的速度
	- 通过固定壁面的通量为零
	- 压力梯度为零

**可压缩流动**

基本思路是一样的，但是跨音速和超音速的情况比较特殊。

湍流的边界条件也比较特殊，这些都需要特别的查询专业资料。

## 8. 代数系统

无论如何，离散方程组最后都能整理为

$$
a_P\phi_{P}^{n}+\sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P}
$$

- 对流项考虑 CD / UD / BD 等插值格式
- 扩散项考虑梯度插值计算和非正教修正
- 源项线性化

对于每一个控制体都会有这么一个本地组装的离散方程，形成一个线性代数系统

$$
[A]\bigg[\phi\bigg] = [R]
$$

- 矩阵 $[A]$ 是一个稀疏矩阵，$a_P$ 在对角线上，$a_N$ 在对角线旁。
- 矩阵 $\bigg[\phi\bigg]$ 是所有计算域内控制体的待求体中心值形成的向量
- 矩阵 $[R]$ 是源项向量，包含所有不含 $\phi$ 的项、含有 $\phi$ 旧时间步的所有项
- 系数 $a_P$ 部分既有来自时间项，也来自对流项扩散项，也来自源项的线性部
- 系数 $a_N$ 部分来自每一项计算中涉及到的邻单元部分

当这个线性代数系统被求解后，我们就获得了新时间步下的物理场 $\phi$（网格单元中心值）。

现有的代数解法分成直接法和迭代法。通常来说，当矩阵规模增大的时候，迭代法更加经济，但是也对矩阵的特性有一些要求。

### 8.1. 矩阵性质

上面的 LHS 矩阵是稀疏的，大多数的矩阵元素都等于零，如果求解器可以利用稀疏特性，将大大降低计算机的内存需求。

当对角线上元素等于非对角线上元素之和，称为 diagnally equal 。对角线元素大于非对角线元素值和的时候，称为 diagnal dominance ，即对角占优。至少有一行满足对角占优的矩阵称为对角占优矩阵。

迭代法需求矩阵的对角占优来保证收敛。

源项的离散和对角占有有直接关系。

回忆 RHS 的源项线性化如下

$$
S_{\phi}(\phi) = Su + Sp\phi
$$

如果 $S_{p} < 0$ ，移项到 LHS 的时候，增加了 LHS 的矩阵对角占优，也就增加了迭代计算收敛性。

如果 $S_{p}> 0$ ，移项到 LHS 的时候，减小了 LHS 的矩阵对角占优，也减弱了计算收敛性，这个时候应该把它显式离散，留在 RHS。

对流项的离散格式都可以看成是 UD 格式的升级。离散的时候，可以选择性的进行隐式离散（留在 LHS 矩阵）或者显式离散（留在 RHS 矩阵）。

扩散项的离散在正交网格下生成 diagonally equal 矩阵，但是网格的非正交性破坏了这点，也破坏了有界性。一般来说，扩散项的非正交修正部分与隐式部分（正交影响部分，离散到 LHS）相比较小。所以正交影响部分隐式处理，留在 RHS，生成 diagonally equal 矩阵。而非正交修正部分则显式处理，移到 RHS，加入广义源项。 如果非正交影响较大，则需要加入非正交修正器，额外使用迭代来计算非正交修正项，每次迭代都要更新非正交修正项，直到误差满足要求。 这些分开的隐式显式的处理只能提高矩阵性质，并不能保证有界性。 

### 8.2. 欠松弛

实践证明，源项的线性部分和时间项能实际上可以加强 LHS 矩阵的对角占优性质。

在稳态计算中，没有时间项离散对 LHS 矩阵对角占优性质的加强，可以使用欠松弛技术。

考虑离散后得到的线性代数系统

$$
a_P\phi_{P}^{n}+\sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P}
$$

为该式增加人工项

$$
a_P\phi_{P}^{n}+ \bigg(\frac{1-\alpha}{\alpha}a_P\phi_{P}^{n}\bigg) + \sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P} + \bigg(\frac{1-\alpha}{\alpha}a_P\phi_{P}^{0}\bigg)
$$

- $0<\alpha\leq 1$ 是欠松弛因子
- 当问题达到稳态结果的时候，$\phi_{P}^{n} = \phi_{P}^{0}$ 

上式整理为

$$
\frac{a_P}{\alpha}\phi_{P}^{n}+\sum\limits_{N}a_{N}\phi_{N}^{n} = R_{P} + \bigg(\frac{1-\alpha}{\alpha}a_P\phi_{P}^{0}\bigg)
$$

因为 $\alpha$ 是小于 1 的，所以上式的对角元素 $\alpha_P$ 的系数得到了放大，也就是增加了对角占优性质。

### 8.3. 代数系统求解

可以使用 CG 方法等算法进行代数求解，暂不展开。

## 9. 小结

在上述讨论中，我们讨论了有限体积法的基本思想，包括四大项的基本离散。相信读者能梳理出从连续的控制方程，到离散的离散方程，到求解线性代数方程组，这个完整的数值过程。更复杂的情况暂不深究。

>[!warning]
>如果有读者困惑为什么离散方程组可以作为线性代数系统进行代数求解，应该简单复习《数值计算》（或《科学计算》等类似教科书），重新理解数值方法的基本思想即可。建议只是适当复习，不应投入大量时间再次重新学习，以后遇到问题反复查阅即可。

此外，因为反复提到 `linear` ，可能有读者困惑这不是重复指定了吗？

其实并不是，可以梳理如下

| 位置                     | 作用对象     | 具体含义        |
| ---

> [!important]
> 访问 [https://aerosand.cc](https://aerosand.cc/) 以获取最近更新。
------------------- | -------- | ----------- |
| `interpolationSchemes` | 所有面上的物理量 | 全局默认的线性插值   |
| `gradSchemes`          | 梯度计算中的面值 | 梯度特定项的线性插值  |
| `divSchemes`           | 对流项中的面值  | 对流项特定项的线性插值 |
| `laplacianSchemes`     | 扩散项中的面值  | 扩散项特定项的线性插值 |



本文完成讨论

- [x] 有限体积法基础
- [x] 了解四大项的离散
- [x] 了解离散方程与 OpenFOAM 代码的部分对应

## References

[1] The Finite Volume Method in Computational Fluid Dynamics, https://link.springer.com/book/10.1007/978-3-319-16874-6

[2] Computational fluid dynamics : the basics with applications, https://searchworks.stanford.edu/view/2989631

[3] Mathematics, Numerics, Derivations and OpenFOAM®, https://holzmann-cfd.com/community/publications/mathematics-numerics-derivations-and-openfoam-free

[4] Notes on Computational Fluid Dynamics: General Principles, https://doc.cfd.direct/notes/cfd-general-principles/

## 支持我们

>[!tip]
>希望这里的分享可以对坚持、热爱又勇敢的您有所帮助。
>
>如果这里的分享对您有帮助，您的评论或赞助将对本系列以及后续其他系列的更新、勘误、迭代和完善都有很大的意义，这些行动也会为后来的新同学的学习有很大的助益。
>
>赞助打赏时的信息和留言将用于展示和感谢。

{{< cards >}}
  {{< card link="/" title="支持" image="https://www.notion.so/image/attachment%3A3be6af9a-4829-4dfd-997e-641dfd055ba9%3Aalipay.jpg?table=block&id=22cd34b0-7c4c-8086-bdda-d558df1d9a11&t=22cd34b0-7c4c-8086-bdda-d558df1d9a11" subtitle="支付宝" >}}
{{< /cards >}}

> Copyright @ 2026 Aerosand
>
> - 课程（文本、图片等）：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
> - OpenFOAM 开发代码：[GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)
> - 其他代码：[MIT License](https://opensource.org/licenses/MIT)
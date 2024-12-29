---
title: 关于代数基本定理
date: 2020-11-20 20:45:05
tags: [math]
---

### 定理描述

> 任一次数 $\ge1$ 的复多项式在复数域中至少有一个根。

### Gauss 的证明方法

代数基本定理的第一个实质性证明是由 Gauss 在他的博士论文中给出的。

设 $f(x)\in \mathbb{C}[x]$ 且 $\partial(f(x))\ge 1$，那么 $f(x)$ 有一次因式，设为 $x-\alpha_1$，令 $x-\alpha_1$在 $f(x)$ 中是 $l_1-$ 重因式，则 $(x-\alpha_1)^{l_1}|f(x)$，令

$$
f_1(x)=\frac{f(x)}{(x-\alpha_1)^{l_1}}
$$

若 $\partial(f_1(x))\ge 1$，则同理，$f_1(x)=\frac{f(x)}{(x-\alpha_1)^{l_1}}$ 也有一次因式 $x-\alpha_2$，设其重数为 $l_2$，则 $(x-\alpha_2)^{l_2}|f_1(x)$，依次得

$$
f(x),f_1(x)=\frac{f(x)}{(x-\alpha_1)^{l_1}},\cdots,f_t(x)=\frac{f_{t-1}(x)}{(x-\alpha_t)^{l_t}}
$$

使得

$$
\partial(f_t(x))=0,f_t(x)=a_n
$$

因此，

$$
f(x)=a_n(x-\alpha_1)^{l_1}(x-\alpha_2)^{l_2}\cdots(x-\alpha_t)^{l_t}
$$

其中，$\alpha_1,\cdots,\alpha_t$ 是不同的复数，$l_1,\cdots,l_t$ 是正整数，且 $l_1+\cdots+l_t=\partial(f(x))$。

这是 $f(x)$ 的标准分解，从而有，

1. 复数域上每个次数 $\ge1$ 的多项式都可以唯一地分解为一次因式的乘积；

2. 复数域上每个 $n$ 次多项式都恰有 $n$ 个复根（重根按重数记）；


### 衍生

代数基本定理表明复数域作为一个数系是完善的。


设 $f(x) = a_nx^n + a_{n-1}x^{n-1} + \cdots + a_0 \in \mathbb{R}[x]$，$f(x)$ 至少有一个复根，设为 $\alpha$，即

$$
f(\alpha)=a_n\alpha^n+a_{n-1}\alpha^{n-1}+\cdots+a_0=0
$$

取共轭，有

$$
f(\bar\alpha)=a_n\bar\alpha^n+a_{n-1}\bar\alpha^{n-1}+\cdots+a_0=0
$$

这说明 $\bar\alpha$ 也是 $f(x)$ 的一个根。

如果 $\alpha$ 是一个实数，那么 $\alpha=\bar\alpha$ 且 $x-\alpha$ 是 $f(x)$ 的一个一次因式.

如果 $\alpha\in\mathbb{C}$ 不是实数，那么 $\alpha\ne\bar\alpha$，从而 $x-\alpha$ 和 $x-\bar\alpha$ 是 $f(x)$ 的两个不同的复一次因式，因为

$$
(x-\alpha,x-\bar\alpha)=1
$$

所以，在 $\mathbb{C}$ 上，$(x-\alpha)(x-\bar\alpha)|f(x)$，因为

$$
(x-\alpha)(x-\bar\alpha)=x^2-(\alpha+\bar\alpha)x+\alpha\bar\alpha
$$

是一个实二次多项式，从而在 $\mathbb{R}$ 上也有 $(x^2-(\alpha+\bar\alpha)x+\alpha\bar\alpha)|f(x)$ 成立，又由因式分解的唯一性，

$$
x^2-(\alpha+\bar\alpha)x + \alpha\bar\alpha
$$

是实不可约多项式，于是有

**命题** 一个实多项式 $f(x)$ 的非实复数根总是成对出现，即当 $\alpha$ 是 $f(x)$ 的根的时候，$\bar\alpha$ 也是 $f(x)$ 的根，且 $x^2 - (\alpha + \bar\alpha)x + \alpha\bar\alpha$ 是 $f(x)$ 在实数域上的不可约二次因式。

每个次数 $\ge 1$ 的实多项式在实数域上总可以唯一地分解为一次因式和二次不可约因式的乘积。


复不可约多项式是一次的，实不可约多项式是一次或者二次的，那么有理不可约多项式呢？

有理多项式的因式分解可以归结为整数多项式的因式分解。

**Gauss 引理** 两个本原多项式的积仍然是本原多项式。

**证**：

设

$$
\begin{aligned}
f(x) & = a_nx^n + \cdots + a_1x + a_0,\\
g(x) & = b_mx^m + \cdots + b_1x + b_0
\end{aligned}
$$

是两个本原多项式，令

$$
h(x)\triangleq f(x)g(x)=d_{n+m}x^{n+m}+\cdots+d_1x+d_0
$$

其中，$d_l=\sum_{s+t=l}a_sb_t$，对于 $l=0,1,\cdots,n+m$。

$f(x)$ 和 $g(x)$ 是本原的，故 $p$ 不能整除它们所有的系数，从而存在 $i$ 和 $j$，使得

- $p|a_0,\cdots,p|a_{i-1}$ 但是 $p\nmid a_i$；

- $p|b_0,\cdots,p|b_{j-1}$ 但是 $p\nmid b_i$；

考虑

$$
d_{i+j}=a_ib_j+(a_{i+1}b_{j-1}+a_{i+2}b_{j-2}+\cdots+a_{i+j}b_0)+(a_{i-1}b_{j+1}+a_{i-2}b_{j+2}+\cdots+a_{0}b_{i+j})
$$

其中 $p|d_{i+j}$，从而也整除右边，但 $p$ 实际上整除右边除了 $a_ib_j$ 外的所有项，所以 $p$ 不能整除右边，矛盾.

所以 $h(x)$ 是本原多项式.

**定理** 一个非零的整系数多项式若能分解成两个较低次有理多项式之积，则一定也能分解为两个较低次整系数多项式之积.

**证**:

设整系数多项式

$$
f(x)=g(x)h(x)
$$

其中 $g(x),h(x)\in\mathbb{Q}[x]$ 且 $\partial(g(x))<\partial(f(x))$，$\partial(h(x))<\partial(f(x))$，则存在本原多项式 $f_1(x),g_1(x),h_1(x)$ 使得

$$
f(x)=af_1(x),\ g(x)=rg_1(x),\ h(x)=sh_1(x)
$$

其中 $a\in \mathbb{Z}$，$r,s\in\mathbb{Q}$，从而 $af_1(x)=rsg_1(x)h_1(x)$。

由 Gauss 引理，$g_1(x)h_1(x)$ 是本原多项式，$f_1(x)=\pm g_1(x)h_1(x)$，$a=\mp rs$，则 $rs\in \mathbb{Z}$，于是

$$
f(x)=(rsg_1(x))h_1(x)
$$


其中 $rsg_1(x)$ 是整系数多项式.

类似的，也可以得到

**命题** 设 $f(x),g(x)$ 是整系数多项式且 $g(x)$ 是本原的，若 $f(x)=g(x)h(x)$，其中 $h(x)$ 是有理多项式，那么 $h(x)$ 必是整系数的.

然后就有一个求整系数多项式全部有理根的方法，也就是

**定理** 设 $f(x)=a_nx^n+\cdots+a_1x+a_0$ 是一个整系数多项式，$a=\frac{r}{s}$ 是 $f(x)$ 的一个有理根，其中 $r$ 和 $s$ 是互素的整数，那么一定有 $r|a_0$，$s|a_n$.

**证**：

因为 $f(\alpha)=0$，所以在 $\mathbb{Q}$ 上有 $(x-\alpha)|f(x)$，从而 $(sx-r)|f(x)$.

由 $s$ 与 $r$ 互素，$sx-r$ 是本原多项式，于是有整系数多项式 $b_{n-1}x^{n-1}+\cdots+b_1x+b_0$ 使得

$$
f(x)=(sx-r)(b_{n-1}x^{n-1}+\cdots+b_1x+b_0)
$$

比较上式两边的整系数，得 $a_n=sb_{n-1}$，$a_0=-rb_0$.

所以 $r|a_0$，$s|a_n$.

特别的，若 $f(x)=x^n+a_{n-1}x^{n-1}+\cdots+a_1x+a_0$ 是一个首系数为$1$得整系数多项式，那么 $f(x)$ 可能的有理根必为整数且均为 $a_0$ 的因子.

**Eisenstein 判别法** 设 $f(x)=a_nx^n+a_{n-1}x^{n-1}+\cdots+a_1x+a_0x$ 是一个整系数多项式，如果存在素数 $p$ 使得：

1. $p\nmid a_n$

2. $p|a_{n-1}, a_{n-2}, \cdots, a_0$

3. $p^2\nmid a_0$

那么 $f(x)$ 在有理数域上是不可约多项式.

**证**:

用反证法，若 $f(x)$ 在 $\mathbb{Q}$ 上可约，则 $f(x)$ 可以分解为两个较低次整系数多项式之积，设为

$$
f(x)=(b_sx^s+\cdots+b_1x+b_0)(c_tx^t+\cdots+c_1x+x_0)
$$

则有 $a_n=b_sc_t$，$a_0=b_0c_0$。

一般地，对于 $0\le k\le n$，有

$$
a_k=b_kc_0+b_{k-1}c_1+\cdots+b_1c_{k_1}+b_0c_k
$$

因为 $p|a_0$，所以 $p|b_0$ 或者 $p|c_0$，但 $p^2\nmid a_0$，于是 $p|b_0$ 和 $p|c_0$ 不同时成立，不妨设 $p|b_0$ 但 $p\nmid c_0$.

另一方面，$p\nmid a_n$ 故 $p\nmid b_s$，假设 $b_0,b_1,\cdots,b_s$ 中第一个不能被 $p$ 整除的是 $b_i$，那么 $0\le i\le s<n$，但是

$$
a_i=b_ic_0+b_{i-1}c_1+\cdots+b_0c_i
$$

其中 $a_i,b_0,b_1,\cdots,b_{i-1}$ 均可被 $p$ 整除，故 $p|b_ic_0$，得 $p|b_i$ 或者 $p|c_0$，这与假设矛盾.

需要注意的是，Eisenstein 判别法只是给出了多项式不可约的一个充分而非必要的条件，即就算找不到合适的 $p$ 使条件成立，也不能说多项式一定不可约.

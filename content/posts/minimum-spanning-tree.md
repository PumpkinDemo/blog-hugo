---
title: 最小生成树
date: 2021-08-19 20:05:58
tags: [algorithm]
---

### Prim Algorithm

How does it run?

+ Choose an arbitrary vertex $v$ and let $S=\{v\}$
+ Choose a vertex $u$ closest to $S$ and add it into $S$ and record the corresponding edge
+ go on until all vertices are in $S$
+ The recorded edges and $S$ constructs a **Minimum Spanning Tree**

Let $G=(V,E,W)$ be a weithted connected graph and has $n$ vertices.

$T$ is a tree built through Prim Algorithm.

Show $T$ is a **minimum spanning tree**:

First, there must be an optimal solution which contains the edge with the minimal weight.

If not, we can add this edge into the solution and obtain a cycle, then remove an edge with larger weight and we can obtain a better solution.

Recursively, if $e_1,e_2,\cdots,e_k$ are in a part of some optimal solution, then $e_{k+1}$ is also in it.

</br>

### Kruskal Algorithm

The main idea of Kruskal Algorithm is linking different connected components continuously with the edge with minimal weight, until there is only one connected componen remains.

Procedure:

+ Each vertex constructs a single connected component
+ linking
+ finish

**Rightness:**

Denote the result of Kruskal Algorithm by $T$ and the Minimal Spanning Tree by $T_{min}$

Assume that there are $k$ different edges between $T$ and $T_{min}$ (permutated by weight increasingly)
$$
\begin{aligned}
T:& a_1,a_2,\cdots,a_k\\\\
T_{min}:& b_1,b_2,\cdots,b_k
\end{aligned}
$$
Add $a_1$ into $T_{min}$ and it leads to a cycle including some of $b_1,b_2,\cdots,b_k$

Remove the most weighted edge $b$ in this cycle and belonging to $b_1,b_2,\cdots,b_k$

Then obtain a new spanning tree $T_1$

Edge $a_1$ and edge $b$ must have the same weight

+ Since $T_{min}$ is the **minimum spanning tree**, $W_{a_1}\ge W_b$
+ Since $b$ is not in $T$, according to the rule of **Kruskal Algorithm**, there are 2 cases:
    + $b$ Leads to cycle with vertices added already (since $a_1$ has the minimal weight in $a_i$s, those vertices should be the common vertices of $T$ and $T_{min}$), but is impossible because $b$ is exactly in $T_{min}$
    + or the second case, the weight of $b$ is no less than $a_1$ and would be considered after $a_1$
+ Then it's clear, $W_{a_1}=W_b$

And $T_1$ is also a **minimum spanning tree**

Following this way, we can convert $T_{min}$ into $T$ with keeping its weight

Hence $T$ is also a **minimum spanning tree**

</br>

### Intresting

MST is the short of **"Minimum Spanning Tree"**.

Why not **"Minimal Spanning Tree"**?

One kind of explanation:

+ **minimum** is a *quantitative* representation of the smallest amount needed
+ **minimal** is a is a *qualitative* characteristic


---
date: '2020-12-28T17:16:08+08:00'
draft: false
title: Test
toc: true
---

## Introduction

This is **bold** text, and this is *emphasized* text.

## Test for Equations

This is an inline equation: $a + b = ?$

This is a display equation:

$$
\begin{aligned}
KL(\hat{y} || y) &= \sum_{c=1}^{M}\hat{y}_c \log{\frac{\hat{y}_c}{y_c}} \\
JS(\hat{y} || y) &= \frac{1}{2}(KL(y||\frac{y+\hat{y}}{2}) + KL(\hat{y}||\frac{y+\hat{y}}{2}))
\end{aligned}
$$

## Test for Code Block

```python
import os

os.system("cat /flag")
```

Visit the [Hugo](https://gohugo.io) website!

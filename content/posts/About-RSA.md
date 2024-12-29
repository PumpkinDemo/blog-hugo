---
title: About RSA
date: 2021-10-30 16:12:25
tags: [Crypto]
---

### RSA procedure

1. choose two primes, $p$ and $q$
2. calculate $N = p\times q$
3. calculate $r = \varphi(N)= \varphi(p\times q)=(p-1)(q-1)$
    + $\varphi$ is the Eular function, $\varphi(n)$ represents the number of integers prime with $n$
    + if $n$ is a prime, then obviously $\varphi(n)=n-1$
    + $\varphi(mn)=\varphi(m)\varphi(n)$
4. choose $e$ no more than $N$ and prime with $r$
5. calculate $d$ such that $ed \equiv1 \mod r$
6. then $(e, N)$ is public key and $(d, N)$ is private key
7. don't forget to destroy $p$ and $q$

Now Alice wants to send some message to Bob.

She needs to transform the message string into a integer $m$ no more then $N$.

+ ciphertext $c = m^e \bmod N$

When Bob receive the ciphertext, he needs to decrypt it.

+ plaintext $m = c^d \bmod N$


### Rightness

How to show that Bob can get right raw message?

Just to show that $c^d \equiv m \mod N$
$$
\begin{aligned}
c^d &\equiv (m^e)^d \mod N \\
&\equiv m^{ed} \mod N \\
&\equiv m^{tr+1} \mod N \\
&\equiv m\cdot m^{tr} \mod N \\
&\equiv m\cdot (m^{\varphi(N)})^t \mod N \\
&\equiv m\cdot 1^t \mod N \\
&\equiv m \mod N
\end{aligned}
$$
The key point is $m^{\varphi(N)}\equiv1 \mod N$ when $\gcd(m, N)=1$


### More

In fact, we can choose more than two primes at beginning.

For example, choose $n$ primes $p_1, p_2, \cdots,p_n$

calculate $N = p_1\times p_2\times\cdots\times p_n$

and calculate $r = \varphi(N) = (p_1-1)(p_2-1)\cdots(p_n-1)$

then everything is the same.



The difficulty to crack RSA decides to the difficulty to factoring a big intger.

The big intger is just $N$ and if we can factor it into some primes, then we can calculate $r$, and get $d$.
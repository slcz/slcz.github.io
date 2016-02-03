---
layout: post
title:  "Write Amplification, Randomness and Math"
date:   2016-02-02 13:15:31 -0800
categories:
tags: ["SSD", "Flash", "Write Amplification"]
---

<style>
    table.numbered-equation { width:99%;  text-align: center; vertical-align: middle;
        margin-top:0.5em; margin-bottom:0.5em; line-height: 2em; font-size: 60%; }
    td.equation-number { text-align:right; width:2em; }
</style>

When a block is finally selected for garbage collection, we could calculate
the probability that it is not overwritten by a newer write. As below,

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            P_{valid}=(1-\frac{1}{U})^W
            </script>
        </td>
        <td class="equation-number">
            (1)
        </td>
    </tr>
</table>
Where,

- $$P_{valid}$$ is the probability that the block is still valid
- $$U$$ is the total amount of useable blocks
- $$W$$ is the total number of of younger writes coming from external user

Due to lazy GC, $$U, W$$ are both very large numbers that are in the same
magnitude as the total physical capacity (millions to billions of blocks).
We further introduce 2 variables, $$f$$ and $$\lambda$$. $$f$$
 is the ratio of useable capacity to physical capacity. $$\lambda$$,
defined as the ratio of external writes to total device writes, is the
inverse of write amplification, $$\omega$$.

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            U=fS
            </script>
        </td>
        <td class="equation-number">
            (2)
        </td>
    </tr>
</table>

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            W=\lambda S=\frac{S}{\omega}
            </script>
        </td>
        <td class="equation-number">
            (3)
        </td>
    </tr>
</table>

From _Eq._ (1), (2) and (3), we get,

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            P_{valid}=(1-\frac{1}{U})^W=
            (1-\frac{\lambda}{f}\frac{1}{\lambda S})^{\lambda S}=
            (1-\frac{\lambda}{f}\frac{1}{W})^W
            </script>
        </td>
        <td class="equation-number">
            (4)
        </td>
    </tr>
</table>

In the case where garbage collectio is lazy, $$W$$ is large. As $$W$$
approaches infinity, the page valid probability becomes,

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            \Psi=\lim_{W \to \infty} P_{valid}=\lim_{W \to \infty}(1-\frac{\lambda}{f}\frac{1}{W})^W=e^{-\frac{\lambda}{f}}
            </script>
        </td>
        <td class="equation-number">
            (5)
        </td>
    </tr>
</table>

$$\Psi$$ is approximately the oldest blocks written. _Eq._ (5) is due to the fact that
<script type="math/tex"> \lim_{n \to \infty} (1+\frac{k}{n})^n = e^k</script>.
Further, write amplification has the following relations to the probability of
a block being valid,

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            \omega=\frac{1}{\lambda}=\frac{1}{1-\Psi}
            </script>
        </td>
        <td class="equation-number">
            (6)
        </td>
    </tr>
</table>

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            \Psi=1-\lambda
            </script>
        </td>
        <td class="equation-number">
            (7)
        </td>
    </tr>
</table>

Put _Eq._ (5) and (7) together,

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            1-\lambda=e^{-\frac{\lambda}{f}}
            </script>
        </td>
        <td class="equation-number">
            (8)
        </td>
    </tr>
</table>

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            1-\frac{1}{\omega}=e^{-\frac{1}{f\omega}}
            </script>
        </td>
        <td class="equation-number">
            (9)
        </td>
    </tr>
</table>

Where,

- $$\omega$$ is the write amplification,
- $$f$$ is the ratio of physical capacity to useable capacity.

Notice _Eq._ (9) does not hae elementary solution, but it can be solved by
using [lambertW function][lamberW].

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            \omega=\frac{\zeta}{\zeta-W(\zeta e^\zeta)}
            </script>
        </td>
        <td class="equation-number">
            (10)
        </td>
    </tr>
</table>

<table class="numbered-equation" cellpadding="0" cellspacing="0">
    <tr>
        <td>
            <script type="math/tex; mode=display">
            \zeta=-\frac{1}{f}
            </script>
        </td>
        <td class="equation-number">
            (11)
        </td>
    </tr>
</table>

_Eq._ (10) and (11) solves write amplification as a function of useable 
ratio. With a good [LambertW function approximator][approx], we can
calculate,

| useable-ratio | write-amplification |
|--------------:| ------------------: |
| 95%           | 10.17               |
| 90%           | 5.18                |
| 85%           | 3.52                |
| 80%           | 2.69                |
| 75%           | 2.20                |
| 70%           | 1.88                |
| 65%           | 1.65                |
| 60%           | 1.48                |
| 55%           | 1.35                |
| 50%           | 1.26                |

Or plot,

<img alt="write amp to useable ratio" src="/assets/wa.png"/>

[lamberW]: https://en.wikipedia.org/wiki/Lambert_W_function
[approx]:  http://keithbriggs.info/software/LambertW.c
[w2f]:     /assets/wa.png

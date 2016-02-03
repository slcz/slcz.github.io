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

Imagine sitting inside a sushi restraunt, where shshi plates are placed on a slowly moving conveyor bar. Customer picks up full plate and leaves empty one on the bar. Suppose a waiter is standing at the end and moving empty plates to the kitchen, meanwhile keeps adding new plates into the loop. How fast would the waiter moving plates in order to keep the bar full without leaving empty plates on the bar for too long?

While testing on SSDs, I encountered the same question many times.
In a somewhat similar situation as the sushi bar metaphor, a flash device is written in a sequtial manner regardless of incoming write's address pattern. Considering a simplified case, suppose external users (applications) write to the device consistently and the workload is randomly distributed across the entire usable space. How fast should the SSD move fragmented blocks around in order to keep up with the data coming rate?

All the SSDs expose a smaller capacity to the applications than the physical capacity. We call the difference "space overprovisioning", for a given workload, the more the reservation, the less work flash's garbage collector has to do to repack the working set.

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

- $$P_{valid}$$ is the probability that the block is still valid when it's
scanned for GC.
- $$U$$ is the total amount of usable blocks
- $$W$$ is the total number of of younger writes coming from external user

Due to lazy GC, $$U, W$$ are both very large numbers that are in the same
magnitude as the total physical capacity (millions to billions of blocks).
We further introduce 2 variables, $$f$$ and $$\lambda$$. $$f$$
 is the ratio of usable capacity to physical capacity. $$\lambda$$,
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
            (1-\frac{\lambda}{f}\frac{1}{\lambda S})^{W}=
            (1-\frac{\lambda}{f}\frac{1}{W})^W
            </script>
        </td>
        <td class="equation-number">
            (4)
        </td>
    </tr>
</table>

In the case where garbage collection is lazy, $$W$$ is large. As $$W$$
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

$$\Psi$$ approximates the valid possiblity of the earliest block written in the system. _Eq._ (5) is due to the fact that
<script type="math/tex"> \lim_{n \to \infty} (1+\frac{k}{n})^n = e^k</script>.
Further, write amplification has the following relationship to the block valid probability,

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
- $$f$$ is the ratio of physical capacity to usable capacity.

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

_Eq._ (10) and (11) solves write amplification as a function of usable 
ratio. With a good [LambertW function approximator][approx], we can
calculate,

| usable-ratio | write-amplification |
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

<img alt="write amp to usable ratio" width="800" src="/assets/wa.png"/>

[lamberW]: https://en.wikipedia.org/wiki/Lambert_W_function
[approx]:  http://keithbriggs.info/software/LambertW.c
[w2f]:     /assets/wa.png

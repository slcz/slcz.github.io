---
layout: post
title:  "Write Amplification, Randomness and Math"
date:   2016-02-02 13:15:31 -0800
categories:
tags: ["SSD", "Flash", "Write Amplification"]
---

Imagine sitting inside a sushi restaurant, where shshi plates are placed on a slowly moving conveyor bar. Customer picks up full plates and leaves empty ones on the bar. Suppose a waiter is standing at the end of conveyor and moving empty plates to the kitchen, meanwhile keeps adding new plates into the loop. How fast should the waiter moves the plates in order to keep the bar full without leaving empty plates on the bar for too long?

<img alt="" width="400" src="/assets/sushi.jpg"/>

From time to time, while working on flash controller and SSDs, I encountered the same question. In a somewhat similar situation as the sushi bar metaphor, a flash device is written sequentially regardless of host writes' address pattern. Considering a simplified case, suppose an external host (applications) write to the device consistently and the workload is randomly distributed across the entire address space. How fast should the SSD move fragmented blocks around in order to keep up with the data ingest rate?

We know a SSD exposes smaller capacity to the applications than its raw capacity. Overtime, a younger write with the same host address may overwrite one or more older ones (to the same address), creating space fragmentation. Therefore, a background garbage collector is running and packs data and defragments the holes. For a given workload, the more the over-provisioning space, the less work SSD's garbage collector has to do to re-pack the working dataset. Obviously, the trade-off of having more space reservation is the wasteful of flash capacity.

Modern flash controller inside SSD may employ sophisticated garbage collection algorithm. Some of them exploit spatial locality, e.g., they may try to pack infrequently accessed data together in order to reduce the write amplification (defined as back-end writes to host writes ratio). For instance, a multiple streams GC algorithm writes cold data into a separate stream.

<img alt="multiple stream GC" width="500" src="/assets/gc.png"/>

While it is understandable that an intelligent algorithm may take advantage of workload patterns, sometimes we are interested in the case where the entire write workload (both in terms of address and data pattern) is randomly distributed. This provides a fair ground when comparing the benchmarks of the I/O throughput. The same applies to data patterns, too. Random pattern is used heavily in micro-benchmarking tools such as [_fio_][fio].

Another point that may not be so obvious is that in terms of distribution of the fragmentation, a randomly generated address pattern is different from a uniformly strided (stride >> 1) address pattern. For write addresses that are generated by a true random generator, an old aged block has better probability of being overwritten by younger ones, while the younger blocks are less likely. A uniformly strided address pattern causes all the blocks on the device to have the same probability of being "staled".

<img alt="fragmentation distribution" width="500" src="/assets/holes.png"/>

In a randomly distributed workload where there is no obvious distinction of address pattern locality, multiple-queue GC algorithm won't work better than a naive single queued, round-robin GC algorithm. Therefore, I choose the simpler GC to model random workload write amplification. Also, I choose to model the GC algorithm as being lazy, i.e., GC only happens when there is no free space left. In another words, gc-threshold above is set close to zero.

<img alt="multiple stream GC" width="500" src="/assets/naive_gc.png"/>

While it is possible to run a simulator and get write amplification distribution on space over-provisioning, wouldn't it be interesting to try to derive the model analytically? We know for any block that carries data, we could calculate the probability that it is still valid (not overwritten). As below,

<table class="numbered-equation">
    <tr>
        <td class="eq">
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

- $$P_{valid}$$ is the probability that the block is valid.
- $$U$$ is the host addressable space (number of blocks).
- $$W$$ is the total number of of younger writes from host

Due to lazy GC policy, $$U, W$$ are both very large numbers that are in the same magnitude as the total physical capacity in the device (millions to billions of 4KB blocks). Further, we introduce 2 variables,
$$f$$ and $$\lambda$$. $$f$$
 is the ratio of usable capacity to physical capacity. $$\lambda$$,
defined as the ratio of host writes to total device writes, is the
inverse of write amplification, which we denoted as $$\omega$$.

<table class="numbered-equation">
    <tr>
        <td class="eq">
            <script type="math/tex; mode=display">
            U=fS
            </script>
        </td>
        <td class="equation-number">
            (2)
        </td>
    </tr>
</table>

<table class="numbered-equation">
    <tr>
        <td class="eq">
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

<table class="numbered-equation">
    <tr>
        <td class="eq">
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

<table class="numbered-equation">
    <tr>
        <td class="eq">
            <script type="math/tex; mode=display">
            \Psi=\lim_{W \to \infty} P_{valid}=\lim_{W \to \infty}(1-\frac{\lambda}{f}\frac{1}{W})^W=e^{-\frac{\lambda}{f}}
            </script>
        </td>
        <td class="equation-number">
            (5)
        </td>
    </tr>
</table>

$$\Psi$$ approximates the valid probability of the earliest block written in the system. _Eq._ (5) is due to the fact that
<script type="math/tex"> \lim_{n \to \infty} (1+\frac{k}{n})^n = e^k</script>.
Also, write amplification has the following relationship to the block valid probability,

<table class="numbered-equation">
    <tr>
        <td class="eq">
            <script type="math/tex; mode=display">
            \omega=\frac{1}{\lambda}=\frac{1}{1-\Psi}
            </script>
        </td>
        <td class="equation-number">
            (6)
        </td>
    </tr>
</table>

<table class="numbered-equation">
    <tr>
        <td class="eq">
            <script type="math/tex; mode=display">
            \Psi=1-\lambda
            </script>
        </td>
        <td class="equation-number">
            (7)
        </td>
    </tr>
</table>

With _Eq._ (5) and (7),

<table class="numbered-equation">
    <tr>
        <td class="eq">
            <script type="math/tex; mode=display">
            1-\lambda=e^{-\frac{\lambda}{f}}
            </script>
        </td>
        <td class="equation-number">
            (8)
        </td>
    </tr>
</table>

<table class="numbered-equation">
    <tr>
        <td class="eq">
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

While _Eq._ (9) does not have elementary solution, it can be solved by
using [lambertW function][lamberW].

<table class="numbered-equation">
    <tr>
        <td class="eq">
            <script type="math/tex; mode=display">
            \omega=\frac{\zeta}{\zeta-W(\zeta e^\zeta)}
            </script>
        </td>
        <td class="equation-number">
            (10)
        </td>
    </tr>
</table>

<table class="numbered-equation">
    <tr>
        <td class="eq">
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
ratio. With a good [LambertW function approximator][approx], I
plug in the numbers and get the following table,

| UsableRatio  |  WriteAmplification  |
|------------- |  ------------------  |
| 95%          |  10.17               |
| 90%          |  5.18                |
| 85%          |  3.52                |
| 80%          |  2.69                |
| 75%          |  2.20                |
| 70%          |  1.88                |
| 65%          |  1.65                |
| 60%          |  1.48                |
| 55%          |  1.35                |
| 50%          |  1.26                |

Also plotted,

<img alt="write amp to usable ratio" width="800" src="/assets/wa.png"/>

This matches with the simulation very well. In reality, most of the
consumer grade SSDs are thinly over-provisioned and leave less than
10% of capacity reservation. This would slow down the performance
by 5X to 10X under stress writes. You could even tell how much space does a
particular SSD preserve by observing their steady state
throughput degradation ratio.
Just don't forget to pre-condition the SSD by filling the SSD
before any stress testing. It may take very long for the performance to stabilize.

The above analytical exercise may also help space and performance planing.
For example, in an anticipated write heavy environment, one could get
predictable throughput by intentionally creating the file system at lower
than advertised capacity.
Next time, format the drive at 70% of the advertised capacity if you want
to ensure the worst case throughput is better than 50% of the peak!

[lamberW]: https://en.wikipedia.org/wiki/Lambert_W_function
[approx]:  http://keithbriggs.info/software/LambertW.c
[w2f]:     /assets/wa.png
[fio]:     http://freecode.com/projects/fio

# How to configure Linux with PREEMPT_RT

With Linux 6.12 PREEMPT_RT real-time configuration has been included into the
mainline kernel enabling real-time scheduling for x86, ARM64 and RISC-V on the
unmodified original Linux kernel. But even for kernel versions >= 6.12 it is
recommened to apply the latest PREEMPT_RT patch, which is actively being
maintained.  For older kernel versions the patches are mandatory. For kernel
versions >= 6.12 the patches add additional architectures and bring several
improvemens (e.g. for systems with accelerated graphics). All new features are
available in the patch queue before they are included into the mainline kernel.
The latest version of the patches is available
[here](https://cdn.kernel.org/pub/linux/kernel/projects/rt/)
and a git tree is available [here](https://git.kernel.org/pub/scm/linux/kernel/git/rt/linux-rt-devel.git).

For production use it is recommended to use the latest available stable version.

# Configuration of the kernel
The most important setting is "Fully Preemptible Kernel" (CONFIG_PREEMPT_RT). On
older kernels this is a setting inside the Preemption Model menu. On recent kernels this
is a separate configuration item under "General Setup". If CONFIG_PREEMPT_RT is
not visible, CONFIG_EXPERT has to be enabled first.
There are a few well known configuration options (usually for debugging) which
can have a high impact on latencies. If you have based your configuration on a
standard distro config, chances are good that one of these options is enabled.
Just to give a few examples of options which have a high impact on latencies and
therefor should be disabled for real-time use cases:

```
DEBUG_LOCKDEP
DEBUG_PREEMPT
DEBUG_OBJECTS
SLUB_DEBUG
```

Building and starting the kernel works similarly to a kernel without PREEMPT_RT
enabled.

# Using a pre-packaged kernel with PREEMPT_RT enabled
Some buildsystems and distributions already offer pre-built binaries and/or recipes
which can be used to take the first steps and to explore how to work with PREEMPT_RT.

## Debian
Debian offers pre-packaged kernels for x86 and ARM64 with PREEMPT_RT enabled.
These can simply be installed with the package manager:

```
# apt-get install linux-image-rt-amd64
```

## Yocto
The Yocto project provides dedicated recipes for kernels with PREEMPT_RT
enabled. The relevant kernel recipe is 'linux-yocto-rt' and the corresponding
image recipe (depending on linux-yocto-rt) is 'core-image-rt'. This image recipe
includes some extra tools, such as rt-tests (described later in this article).
To build an image with linux-yocto-rt you have to add

```
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto-rt"
```

to your local.conf, bblayers.conf or \$MACHINE.conf. If you are creating a new
BSP using linux-yocto-rt by default, you need add this to the \$MACHINE.conf of
the BSP layer and additionally add this line in a bbappend recipe for
linux-yocto-rt:

```
COMPATIBLE_MACHINE:$MACHINE = $MACHINE
```

# Testing the real-time behaviour
After booting the system it is recommended to check if PREEMPT_RT is enabled.
This can be done by

```
$ uname -a
Linux myrtsystem 6.12.16-rt9 #2 SMP PREEMPT_RT Sun Mar 9 23:20:44 CET 2025 x86_64 GNU/Linux
```
and checking for PREEMPT_RT, or by running

```
$ cat /sys/kernel/realtime 
1
```
The expected value in /sys/kernel/realtime is 1.
Now the real-time beaviour can be evaluated. The most commonly used tool for
this purpose is cyclictest (which is part of the [rt-tests project](https://git.kernel.org/pub/scm/utils/rt-tests/rt-tests.git/)). Release tarballes can be found [here] (https://www.kernel.org/pub/linux/utils/rt-tests/). The rt-tests package is available on all major Linux distributions. On a Debian based system it can be installed by running:

```
apt-get install rt-tests
```
cyclictest can run a given number of threads with real-time priority at a given
interval in microseconds. It will keep track of the observed latencies and
report these to a measurement thread. The following example will run one thread
per CPU with SCHED_FIFO 98 at an interval of 250 microseconds. The reported
values are given in microseconds:

```
# cyclictest -S -m -p98 -i250
```

SCHED_FIFO is the scheduling class. The relevant classes for real-time are
SCHED_FIFO and SCHED_RR. SCHED_FIFO works with static priorities (1 is the lowest,
99 is the highest priority). A SCHED_FIFO task keeps running until it gives up
CPU time or it is interrupted by a higher priority task. SCHED_RR works with
the same priorities but tasks with the same priority are scheduled with round
robin. A SCHED_RR task keeps running until it gives up CPU time, it is interrupted
by a higher priority task or a task with the same priority is runnable and the
time slice of the current task is over.
NOTE: A "runaway" real-time task can starve the system. As a protection
mechanism the runtime of real-time tasks can be limited by setting a value in
microseconds in /proc/sys/kernel/sched_rt_runtime_us. The default value is
50ms and the default accounting is per second. This results in 50ms per second
being reserved for non real-time tasks. If the system is running real-time
workloads for 950ms it won't schedule real-time applications for the
next 50ms. This behavior can be disabled by writing -1 to /proc/sys/kernel/sched_rt_runtime_us.

Another method for evaluating the real-time behaviour and to identify possible
sources of latencies is the real-time Linux analysis tool (rtla). Further
information on rtla can be found in the [kernel documentation](https://docs.kernel.org/tools/rtla/index.html). Video recordings of presentations on rtla are avaiable from the [Embedded Linux Conference](https://www.youtube.com/watch?v=-hJ558URAP4) and the [Embedded Open Source Summit](https://www.youtube.com/watch?v=kIMko-07BV0).

# Typical pitfalls

On some Linux distributions a few standard services (like ntpsec) might be
configured with real-time priority by default. Therefor these services can
affect latencies for other real-time tasks. It is recommended to run top and sort by
priority to review all applications with real-time priority.

# Prioritization of important tasks and interrupt threads
After evaluating the configuration the system can be tuned for its specific
use case. A common task is the prioritization of processes and interrupt threads. By
default all interrupt threads will be scheduled with SCHED_FIFO at priority 50.
The following example shows how the interrupt threads for a network card
(enp4s0) and the related NAPI threads can be adjusted:

```
# ps aux | grep enp4s0
[...]
root         658  1.4  0.0      0     0 ?        S    Mar03 312:31 [irq/134-enp4s0-TxRx-0]
root         659  0.0  0.0      0     0 ?        S    Mar03   1:20 [irq/135-enp4s0-TxRx-1]
[...]
root         752  0.0  0.0      0     0 ?        S    Mar03   0:05 [napi/enp4s0-8200]
root         753  0.0  0.0      0     0 ?        S    Mar03   0:03 [napi/enp4s0-8199]
[...]

# chrt -p -f 98 658
# chrt -p -f 98 659
# chrt -p -f 97 752
# chrt -p -f 97 753
```

Further optimization can be done by isolating one or more CPU cores and
reserving these for specific applications. The following example isolates the
cores 2 and 3 (NOTE: A few housekeeping tasks will always stay on isolated
cores) from scheduling and from interrupt routing with a few settings on the
kernel commandline:

```
isolcpus=2,3 rcu_nocbs=2,3 nohz_full=2,3 irqaffinity=0
```

Specific interrupts can now be routed with the smp_affinity setting. The
following example sets the CPU mask for a specific interrupt to CPU 2.

```
echo 4 > /proc/irq/<irq_number>/smp_affinity
```
The related IRQ threads will automatically follow. A very good introduction to core
isolation can be found [here](https://www.suse.com/c/cpu-isolation-introduction-part-1/).

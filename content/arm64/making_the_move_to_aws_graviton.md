---
author: "Syl Taylor"
title: "Making the Move to AWS Graviton (and Arm64)"
date: "2023-07-08"
description: "Navigating the shift to Arm64 compute on the AWS Cloud"
tags: ["arm64", "cloud", "cost optimization", "performance", "sustainability"]
categories: ["arm64"]
hideMeta: true
searchHidden: false
ShowBreadCrumbs: false
ShowToc: true
TocOpen: true
---

**:exclamation:Disclaimer: All opinions are my own.**

## Overview

In simple terms, the **64-bit Arm-based** AWS Graviton is a **general-purpose CPU** (i.e. meant for any workload designed to run on a CPU).

> The CPU is a mandatory part of any compute device and determines which operations (add, multiply, etc.) your machine can run. Compute capability can be supplemented with operations done by GPUs, NPUs, etc., but those are out-of-scope here. While there are e.g. Graviton machines with GPUs, remember that Graviton on its own is just a CPU.

Since **compute is one of the main cost drivers on the cloud**, Graviton was designed by AWS to be cheaper and faster for popular workloads. It is available as an option in Amazon EC2 as well as a wide range of popular AWS services such as Amazon RDS, AWS Lambda, and AWS Fargate.

### It's a CPU. Why do I even need to write this?

Well, Graviton comes with a fundamental difference compared to existing CPUs seen on the cloud. The instruction set architecture (ISA) behind it is arm64-based (as opposed to x86_64 / amd64) and in the software world is known as arm64 / aarch64.

> In short, a compiler changes what it puts in a software binary depending on which machine you run it on (an x86_64-based one or an arm64-based one) or which flags you specify (to point to a different architecture than the host). Once the binary is generated, it is compatible with just that CPU architecture and ISA version (+ the binary file format is compatible with only certain operating systems).

Exercise: explore the binaries available in https://pypi.org/project/numpy/#files and see if you can find one for Linux operating systems that works on Graviton.

> This is why nowadays when we download software we see a list with multiple options. Here's the Go compiler:
> *	go1.20.4.linux-**amd64**.tar.gz (runs only on x86_64-based machines with CPUs from e.g. Intel, AMD)
> *	go1.20.4.linux-**arm64**.tar.gz (runs only on arm64-based machines with CPUs from e.g. Graviton, Ampere)

This introduces 2 phases of work needed to move to Arm64 that I'll explain.
1. **Enablement** - make your software run on arm64
2. **Performance** - determine if arm64 is a better option

:exclamation: Enablement is made easier by CPU architecture emulation (see QEMU emulator), but it incurs a significant performance hit for any compute-bound workloads (e.g. a software build can take 1 hour instead of 6 minutes).

## Why should you use it?

There is an **increasing demand for compute** in modern workloads and organizations have realized there is a need for better financial management of their cloud infrastructure. To help with FinOps goals, Graviton-based compute options offer customers a potential way to **optimize their spending without sacrificing functionality or performance**.

> It has **up to 20% lower cost** (standard pricing) and up to **40% better price-performance** (e.g. from having software running faster). Another important benefit is the **up to 60% less energy consumption** that helps companies reduce their carbon footprint and become more sustainable. Thus, by migrating to Graviton, organizations can demonstrate their commitment to responsible business practices.

**:exclamation: These numbers will differ based on your software and inputs (e.g. data). Run tests to get exact numbers.**

### How about effort?

In short, it's more like an investment.

Even when you change servers or upgrade to newer x86_64-based instances on the cloud, there will be an effort (to validate performance and migrate).

So, there is shorter-term technical work expected (which takes time / incurs costs). But when you prioritize your expensive or at-scale workloads, you are evaluating if your long-term cost savings offset this technical effort.

> As an example, if I run a web services workload at-scale on 1000 small EC2 instances on a 24/7 basis and assume a 1-to-1 performance mapping, then I'd notice the on-demand pricing differences below:

| CPU type *   | Amazon EC2 Instance | Price / year | $ Diff (m6i) | % Diff (m6i) | $ Diff (m6a) | % Diff (m6a) |
| ------------ | ------------------- | ------------ | ------------ | ------------ | ------------ | ------------ |
| Intel        | m6i.large           | $937320      | NA           | NA           | NA           | NA           |
| AMD          | m6a.large           | $843590      | -$93730      | -10%         | NA           | NA           |
| Graviton 3   | m7g.large           | $797160      | -$140160     | -14.95%      | -$46430      | -5.50%       |
| Graviton 2   | m6g.large           | $753360      | -$183960     | -19.62%      | -$90230      | -10.69%      |

"*" - full names omitted for simplicity

:exclamation: Note there are performance differences expected amongst all 4 types of CPUs and Graviton 3 has up to 25% better performance than Graviton 2. **The real cost savings need to factor in performance.**

### On which workloads?

You should favour workloads with newer software versions and those that have some info on arm64 (such as "it's supported") if possible.

However, note that when changing the CPU, the workloads which are most compute-bound are those that are most likely to have a better performance. There's more differences in Graviton instances than just the CPU and some will give you performance improvements over memory-bound workloads as well. But you'd expect to see increased performance primarily on compute intensive applications.

![Workloads](/making_the_move_to_graviton/workloads.png)

* Network-bound workloads also exist

In this abstract example, some typical software components are listed. They use a variety of hardware resources, but when under heavy load and at larger scale (like we'd see in production), they tend to use one resource more than others, hence we'd call them e.g. resource-bound. 

All workloads can be in scope for a Graviton migration. However, I'd expect the blue CPU-bound boxes to potentially yield most benefits (depends on scale), but note that the data storage (like a database) can also result in a good performance and should be tested too.

## How to Run your Software

**:exclamation: To simplify, consider the names "arm64" and "aarch64" to be equivalent**

The ISA defines the compute operations that the processor can run. Therefore, **for software to run, it must first be compatible with the processor's ISA**. In turn, this means the entire software stack starting from the hypervisor up to the user-level applications, must be compatible with the underlying CPU architecture.

![ISA differences](/making_the_move_to_graviton/isa.png)

This is why you’ll notice on GitHub, Issues asking for arm64 support. Here’s an open request for Cloud Native Buildpacks to add a new feature to support arm64 container image builds: https://github.com/buildpacks/lifecycle/issues/435.

Similarly, if you try to run an x86_64/amd64 container image on an Arm64 machine (like a Graviton instance), you will notice an **exec format error** that indicates CPU architecture incompatibility.

![CPU architecture incompatibility](/making_the_move_to_graviton/exec_format_error.png)

**Tip**: Locate arm64 binaries or container images. If they are missing, ask the provider for them. You can also try to build them if you have access to source code or e.g. Dockerfiles. Here’s an example where you can see the arm64/aarch64 naming convention for the Debian OS and the Linux kernel package:

![arm64 naming convention](/making_the_move_to_graviton/arm64_naming.png)

### State of the Ecosystem

Fortunately, the software ecosystem is ongoingly adding support and optimizations for arm64 which means more binaries get published or improved in time. This is partly driven by users of Arm64 machines which made such requests.

Having readily available binaries and in popular repositories makes the user experience easier and lessens the overhead to migrate. In general, **expect better arm64 support in newer software versions**, so upgrades are encouraged when applicable.

A few examples where it helps to check the latest updates that apply to your workloads:
* Some operating systems don’t provide arm64 images for earlier versions. For example, RHEL 8.2 is the minimum supported on Graviton.
* If you have Go source code, building with [Go >= 1.18 can make it faster by up to 20%](https://aws.amazon.com/blogs/compute/making-your-go-workloads-up-to-20-faster-with-go-1-18-and-aws-graviton/).
* Java >= 11 has optimizations for arm64. In addition, by [experimenting with JVMs and flags](https://github.com/aws/aws-graviton-getting-started/blob/main/java.md) per workload, you can get large performance differences.

Note there are other such details that I recommend you check online for your specific workload before or during your arm64 evaluation.

### Handling Dependencies

Arguably, one hard part in a software migration to arm64 could be resolving all software dependencies, especially if there’s many. In an easy scenario, all will have arm64 binaries and a good performance (meaning they’ve been tested and/or optimized by their providers). In other cases, you have a few options:

1.	**Ask for help** to see if providers can offer arm64 support.
2.	**Remove unused** dependencies to reduce technical debt.
3.	**Upgrade** to get arm64 support or better performance.
4.	**Replace** with an alternative supported package/library.
5.	**Rewrite** with a custom dependency that has arm64 support.

Use your workload’s test suite to minimize chances of introducing breaking changes.

## FAQ

### “Is it easy to migrate?”

#### Easier

Cloud providers like AWS offer managed services, such as databases like Amazon RDS, in which the software is deployed and maintained by their engineering teams. Because they handle the part where they make it work on arm64 efficiently, the customer only consumes the service offered without having to think much about the arm64 aspect.

Container images already pre-built for arm64 (such as those tagged with “ARM 64” in container registries like DockerHub) will already work on Graviton nodes.

![Containers registry](/making_the_move_to_graviton/containers.png)

If you have pre-built binaries for arm64 for any software and dependencies (libraries, frameworks, etc.), you will find the migration easy.

![arm64 binaries](/making_the_move_to_graviton/prebuilt_binaries.png)

Code that uses interpreted (e.g. Python, Ruby) or byte-code (e.g. Java) languages that are platform agnostic will be straightforward to migrate as no changes will be required. There is an exception with these when native code or language extensions are used as these leverage low-level languages such as C/C++ which need to be ported and/or re-compiled for arm64 first. Same exception applies to dependencies such as external libraries which for reasons such as performance, might include native code from compiled languages.

#### Harder

Code using compiled languages like C, C++, Go, Rust, etc. will need to be re-compiled, ideally natively on arm64 machines (faster time, lower complexity). While the compilation process will be similar, you can benefit from using other compilation flags when migrating to arm64.

Some code will be CPU architecture specific, such as that which uses [intrinsic functions](https://developer.arm.com/architectures/instruction-sets/intrinsics/) or [assembly](https://developer.arm.com/documentation/102374/0101/Instruction-sets-in-the-Arm-architecture). These code portions would need to be re-written (often manually) to the equivalent version for arm64. However, note there isn’t always a 1-to-1 mapping, so you could consider alternatives such as highly optimized software libraries (an example would be [aws-lc](https://github.com/aws/aws-lc) for cryptographic instructions).

**:exclamation: In all cases, your workload's performance needs to be assessed for improvements/regressions. Having functional and performance testing for your applications can speed up the Q&A testing and production usage adoption.**

### “How do I measure performance?”

Making software work on arm64 is only half of the story. Performance will be the deciding factor whether to migrate to Graviton. The instances are cheaper, so even if performance is the same, there will still be cost savings.

Entire books have been written on the subject of profiling applications, so these are just high-level notes. When you can, use newer versions of software packages as they tend to include optimizations for better performance on arm64. In general, using Graviton is likely to lead to performance improvements with workloads that are compute-bound (using the CPU mostly, rather than e.g. memory, network, disk) and those that are multi-threaded.

* Don’t rely on benchmarks, measure your own workloads

> Benchmarks rarely reflect real-life workloads. They are also not likely to match your workload properties. To know your cost savings / performance, you need to assess your specific workload under production-like conditions (mimic input data, throughput, latency, etc.).

* Identify the right tooling and process

> Each type of workload will need a setup that it’s suitable for it. For example, for a HTTP web server you can perform load testing using tools such as hey, httperf, or JMeter. In addition, decoupling the client from the server will make monitoring of the server-side metrics such as requests/sec easier and more reliable.

* Use comparable instance types

> Start with same instance family, size, and then compare the appropriate CPU generations
>  - For Graviton 2, e.g. compare m6g.large to m5.large (on Amazon EC2)
>  - For Graviton 3, e.g. compare m7g.large to m6i.large (on Amazon EC2)

* Measure the right metrics

>  - Ideally, maximize CPU utilization on the comparable instance types
>  - Look for a meaningful metric to measure the amount of processing that occurred
>  - Examples of such metrics: requests/sec, reads/sec, writes/sec, execution time, etc.
>
>  To put it simply with an ideal example of running algorithm X and where CPU load is at 100%: you could complete 100 executions with x86_64 and 300 executions with Graviton. Thus, comparing the CPU utilization doesn’t help, but comparing the number of executions completed for the algorithm does.

Once the setup is ready and testing is done, there will still be some cases where performance regressions will occur. For example, PostgreSQL is heavily impacted when the binary isn’t compiled with atomic instructions (e.g. LSE on arm64). Use the open-source community help or any support line you have for cloud providers, software vendors, etc.

### "Are there other Arm64 machines?"

Yes

AWS pioneered Arm64-based servers in the cloud when it introduced AWS Graviton in 2018 and designed it to maximize efficiency for real world workloads. However, Arm-based CPUs have been around for decades and are implemented in [over 250 billion devices](https://www.arm.com/company/news/2022/09/redefining-the-global-computing-infrastructure-with-next-generation-arm-neoverse-platforms). They were initially found mostly in the embedded market (e.g. mobiles, IoT), but now also exist in laptops, workstations, and servers. Historically, Arm CPUs have been praised for lower power consumption and simpler hardware, but nowadays they are also high-performing.

For the majority of cases any software running on Graviton will also work on other Arm64 servers. An exception is code using hardware features not supported on other/older machines (e.g. bfloat16 for machine learning was added in Graviton 3).

As with any other hardware (inc. x86_64-based), changing server type or generations can affect your performance, so test for differences. Examples of other cloud-based Arm64-based compute offerings:
* [Azure VMs with Ampere Altra](https://learn.microsoft.com/en-us/azure/virtual-machines/dpsv5-dpdsv5-series)
* [GCP VMs with Ampere Altra](https://cloud.google.com/compute/docs/instances/arm-on-compute)
* [Oracle VMs with Ampere Altra](https://docs.oracle.com/en-us/iaas/Content/Compute/References/arm.htm)
* [Alibaba Cloud VMs with Yitian](https://www.alibabacloud.com/blog/598159)

In addition, some vendors offer on-premises Arm64 servers and workstations (if you're interested in local development). Apple's post-2020 MacBooks with M1 chips which are Arm64-based can also be used for developing and testing software locally (just make sure not to run Mach-O binaries on Linux).

## Summary

For workloads to run on Graviton, they need to be compatible with the arm64 CPU architecture. This includes the operating system, libraries, container images, and pretty much any software installed. In practice, whenever we have platform agnostic code (e.g. pure Python, pure Java) and all arm64 binaries for the software stack (e.g. Python interpreter, JVM), the migration to Graviton will be easy.

:exclamation: Watch out for AMIs with user data, container definition files, DevOps build scripts, etc., as all commands part of these files need to also be compatible with arm64 and the operating system. Examples of where issues can occur are:
* package installations (e.g. `apt install <package>`)
* running x86_64 binaries (e.g. the wrong file for the Go compiler: `go1.20.4.linux-amd64.tar.gz`)
* running code that uses a dependency with low-level code that needs to be built for arm64 (e.g. `npm install canvas` - install additional packages for building from source and it will work).

Analyze performance based on a meaningful metric such as web requests throughput. Note due to the arm64 software ecosystem maturity, it’s likely your workload’s performance will be better when you use newer software versions.

Prioritize workloads one by one starting with those that have biggest impact (e.g. more cost savings) and lowest friction (e.g. newer supported software stack).

## Useful Links

1. Graviton Tech Guide

More software considerations (such as optimizations) available at: https://github.com/aws/aws-graviton-getting-started

2. Graviton Arm64 Demo

Demo of Arm64 software concepts for an easier adoption of AWS Graviton:
https://github.com/syl-taylor/graviton-arm64-demo

3. Arm64 Quick Guide

10 slides with a high-level overview of Arm64 on Cloud: https://github.com/syl-taylor/arm64-quick-guide-for-cloud

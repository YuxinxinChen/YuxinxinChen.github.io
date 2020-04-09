
layout: post
title: "Notes on GPU"
data: 2020-04-08
tags: [experiment, persistent kernel, GPU]
comments: true
share: false
---

This post assumes that you have basic knowledge of persistent kernel on GPU. 
Besides limitations such as "max threads per block" and "max blocks per SM", the number of blocks that can reside on the GPU and run concurrent is usually limited by register usage and shared memory usage of each block . When I was debugging my code which uses persistent kernel on V100, I found something unexpected: The code which can reproduce the unexpected behavior is commited and push to Owens' runtime-paper repo, branch charmap_queue commit: b3978519bd6754861c463418cf1432a7c6e2cf38. Here is the summary of my experiment:

In the experiment, the runtime system is not launched, and only the BFSCTA kernels are launched. I would like to launch BFSCTA kernel with smaller blocks and fetch size, so that those threads are more responsive (high priority queue). At the same time, I would like to launch BFSCTA kernel on a different stream with larger block and fetch size (low priority queue). In other words, I would like to launch a kernel with two configuarations (block and fetch size) on two streams. The two launched configurations will share the GPU and I need all of launched blocks to reside on GPU and to run concurrently. 

Let's define our first configuration of BFSCTA: 
   * **B1**: block size of first launched kernel,
   * **B2**: block size of second launched kernel,
   * **F1**: Fetch size of first launched kernel, 
   * **F2**: Fetch size of secondly launched kernel,
   * **X**: Number of blocks of first launched kernel with configuration (B1, F1),
   * **Y**: Number of blocks of second launched kernel with configuratoin (B2, F2)
We need **X** **B1**-size blocks and **Y** **B2**-size blocks all reside on GPU and max the GPU occupancy. 

Inside the kernel (BFSCTA ), I have two lines of code:
   1. push low priority work to low priority queue (**l**ocal)
   2. push communication to communication queue (**r**emote)  
In the following experiments, `none` indicates we comment out both code 1 and 2, `l` indicates we comment out only code 2, `r` indicates we comment out only code 1. You could think in the way we have three different kernel each has different register and shared memory usage. 

We will fix **B2**, **F1** **F2** and **X** and see how **Y** changes as **B1** changes in three cases: `none`, `l`, `r`. 

![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/persist.png)

I summarize the unexpected observations below:

1. From the top table, the register usage is ordered 'none < l < r', and shared memory usage is ordered 'none = l < r'. In all cases shown in the top table, the limiting factor is register usage, so we can ignore shared memory usage. We would expect that "**Y** in case `r`" < "**Y** in case `l` < "**Y** in case none", i.e. inverse order of register usage. However, from the bottom table, when **B1** equals 256, `l` has the lowest **Y**. 

2. Another unexpected behavior corresponds to the green cells in Table 2. In all these cells, **Y** is lower than expected from calculating SM usage. I looked into how the scheduler actually assigned the SMs to each block, and noticed a discrepency with the documented scheduler behavior. I illustrate the unexpected behavior in following Figure for the case (**B1**=256, **X**=20, `l`), though I note that all the other green cells demonstrate the same unexpected behavior. As seen in the Figure, the SM 2,4,6...40 are unused. 

![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/sms.png)

In contrast, the yellow cells exhibit expected behavior, and the **Y** values are in line with my calcuations (when register usage is 48, a SM can take 2 512-block and 1 256-block, and when register usage is 64, a SM can only take 2 512-block or 1 256-block plus 1 512-block) as the following Figure shows (**B1** = 256, **X**=2, `r`).

![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/sms2.png)

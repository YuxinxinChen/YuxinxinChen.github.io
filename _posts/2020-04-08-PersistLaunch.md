
layout: post
title: "Notes on GPU"
data: 2020-04-08
tags: [experiment, persistent kernel, GPU]
comments: true
share: false
---

This post assumes that you have basic knowledge of persistent kernel on GPU. 
Beside limitation such as max threads allowed per block, max blocks allowed per SM, how many blocks allowed to resident on the GPU thus could run concurrently largely depend on register usage and shared memory usage of each block. When I was debuging my code which uses persistent kernel on V100, I found something weird. The code which can reproduce the weird behavior is commited and push to owens runtime-paper repo, branch charmap_queue commit: b3978519bd6754861c463418cf1432a7c6e2cf38. Here is the summary of what I have experimented.

In the experiment, runtime system is not launched only the BFSCTA kernels are launched. I would like to launch BFSCTA kernel with smaller blocks and fetch size, so that those threads are more responsive (high priority queue). At the same time, I would like to launch BFSCTA kernel on a different stream with larger block and fetch size (low priority queue). In other words, I would like to launch a kernel with two configuarations (block and fetch size) on two streams. The two launched configurations will share the GPU and I need all of launched blocks resident on GPU and run concurrently. 

Let's define our first configuration of BFSCTA: 
	* **B1**: block size of first launched kernel,
	* **B2**: block size of secondly launched kernel,
	* **F1**: Fetch size of first launched kernel, 
	* **F2**: Fetch size of secondly launched kernel,
	* **X**: Number of blocks of first launch with configuration (B1, F1),
	* **Y**: Number of blocks of second launch with configuratoin (B2, F2)

We need **X** **B1** size blocks and **Y** **B2** size blocks all reside on GPU and max the GPU occupancy. 

In side the kernel (BFSCTA ), I have two lines of code:
	1. push low priority work to low priority queue 
	2. push communication to communication queue.  

In the following experiments, `no r + no l` indicates we comment out both two lines, `l` indicates we comment out only second line, `r` indicates we comment out only first line, and `r + l` indicates we didn't comment out any of the two lines. 

We will fix **B2**, **F1** **F2** and **X** and see how **Y** changes as **B1** changes in three cases: `no r + no l`, `l`, `r`. The results of `r + l` is the same as `r`, so we didn't include it here.

![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/persist.png)

From the top table, `r` and `l` use more register and `r` uses more shared memory, and the max active blocks is limited by register usage not shared memory usage. The limiting factor is register usage in all the cases shown in top table. We would expect in the case of `r`, **Y** should be less than that of case `l` and `no l + no r` since `r` uses more registers. However, from the bottom table, **B1** equals 256 case, `l` has the lowest **Y**. 

Another weird thing in the bottom table is **Y**'s value when **B1** equals 512 case and `l` case when **B1** euqals 256 (gree area).  The yellow area, those **Y**'s value is inligned with my calcuations where when register usage is 48, a SM can take 2 512-block and 1 256-block, and when register usage is 64, a SM can only take 2 512-block or 1 256-block plus 1 512-block. The **Y**'s value in the green area is super weird. I looked into how the scheduler assign SM to each blocks and the green area follows the same scheduing pattern. The following figure uses the case (**B1**=256, **X**=20) to illustrate the pattern. Other cases in the green area apply the same pattern.

![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/sms.png)

Note, configuration **B1**=256, **X**=20 will be launched first and then the second configuration. The scheduler will put the 20 block of first configuration on SM 1,3,5 ... 39. However, the scheduler will not schedule the blocks of second configuration on SM 1,2,3 ... 40, though there is space!

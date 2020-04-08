
layout: post
title: "Concurrent Queue"
data: 2019-05-15
tags: [reading notes, Queue, GPU]
comments: true
share: false
---

This queue is motivited by GPU graph algorithm. Normally most graph algorithm can be mapped to a queue structure. Similar idea is to keep a frontier which only contains the active vertices. Esscentially this frontier is a queue and GPU threads are collectively process the elements in this queue in a synchronized style. Now we want to make it more asynchronized by removing the synchronization after each graph primitive (such as advance, filter) and let the threads go independently. The key structure to achieve this is a concurrent queue. Take page rank as example, the queue will be initialized with enqueue all the vertices into the queue. Each GPU threads will take an different work item from the queue, update its rank and propogate its residule to its neighbors by atomic adding residule to its neighbor's residule and enqueu its neighbor into the queue. In this case, each thread is both a consumer (take an work item and update rank) and a producer (enqueue its neighbor's vertices into the queue). Under this framework, we can see this queue has multiple producers and multiple consumer and they are run independently and asynchronously. 

Then I simplfy this to: a concurrent queue having multiple producers and multiple consumers. Producers and consumers, they are different threads for now.

## Queue basic
Since it is a queue, I will have a counter `start` and a counter: `end`, items between `start` and `end` are valid. The `end` should be larger or equal to `start` all the time.

### Producers 
Normally we assume we couldn't predict when and who will insert and for multiple producer, otherwise we would just ask them to write into an array based their index. We don't want their insertion be overwritten by other producers and idealy their insertion should be compact in terms of memory (there shouldn't be gap between start of the queue and end of the queue in the cases like graph algorithms).   

To enable multiple producers to insert the queue, we introduce another three counters: `end_alloc`, `end_max`, `end_count`. 
### GPUDirect RDMA
From Kepler, GPUDirect RDMA relies on the standard PCIe capabilities to perform peer-to-peer DMA across PCIe devices without CPU involvement. For example, one GPU may read from the memory of another GPU over PCIe, as long as the memory-mapped I/O region exposing the memory of the first is mapped into the address sapce of the second. These capabilities, combined with the GPUDirect RDMA API for GPU memory translations and mappings, enable any PCIe device to access GPU memory in the same way it accesses the CPU memory. 
### Infiniband and HCA
HCA works as a DMA to free the data path from OS involvement. In order to use the network, a process ask the HCA to create a Queue Pair (QP) which has a send work queue and a receive work queue. The send work queue allows the process to be the initiator of transport operations by writing into the entries a Work Queue Element (WQE) describing the send task. The receive work queue allows the process to be a target of a transport operation by writing into the entries a Receive Queue Element (RQE). Each QP has it ID. Each QP, there is a corresponding Completion Queue Element (CQE) which delivers a Compiletion nitification to the initiating process with the status of the operation. We can poll on the CQE to see if the operation is done. So when two processes create a transport connection by instruction the HCA to pair their local QP, as the below shows. So all the information are set up, paired WQE, RQE, send queue, receive queue, place to see send status, and sent data inforamtion is stored in WQE. Where is the buttom we coud press to send? This process is called ring the doorbell. There is a doorbell register in HCA for each QP. The write to the doorbell notifies the HCA to handle the next work request by advancing to the next WQE in the QP.
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/HCA.jpg)

To achive above steps, there is one more thing needs to be set up. The doorbell is on HCA's memory, but the data buffer which will be packed and send out onto network, the QP are on CPU memory. So the data buffer, QP need to be mapped onto HCA's virtual memory system via an operation called Memory registration and a user process performs Infiniband transport operations using VERBs which is the API of Infiniband protocal. The doorbell on HCA also need to be mapped onto CPU's virtual address space and accessed by MMIO as the below figure shows:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/cpuHCA.jpg)

To let the GPU communicate with HCA, we can map GPU's data buffer and QP onto HCA's virtual memory space and map HCA's doorbell onto GPU, then same process happens between CPU and HCA happens between GPU and HCA. GPU fills the information by writing into WQE and poll on CQE, and ring the doorbell when the time to send. As the below figure shows:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/GPUHCA.jpg)

### GPUrdma design decision
1. For single GPU thread, it keep sending requests without worrying about its completion and it only poll on CQ when the send queue is full or all the messages are sent out. Only ring the doorbell when the QP is full.
2. Increases the number of QPs to leverage multiple GPU threads to concurrently create multiple transfer requests. It is actually a widely used techniq to optimize CPU tranfer performance. Then there are some decision the programmer need to make to cater the application:
	1. QP per warp: Good: Packet ordering rules of the RDMA transport protocal force the HCA to serialize the processing of packets that belong to the same QP. Increasing the number of QPs allows the HCA to process different packets from different QPs in parallel.
			Bad: high memory comsuption
	2. QP per threadblock: Good: less memory comsuption
			       Bad: less parallel on HCA.

3. Where to put the QP/CQ. If the QP/CQ on CPU, it is faster for HCA to access but slower for GPU to write into. If it is on GPU, it is faster for GPU to write into, but slower for HCA to access (peer to peer access). But the experiments show it is better to put both on GPU memory.


### Limitation
It is ok:
```bash
	send_A 
	wait_send_A_complete
	send_B
	wait_send_B_complete
```
On remote machine:
```bash
	get_A
	Operate_on_A
	get_B
	Operate_on_B
```
If we send A and B asynchronousely:
```bash
	send_A
	send_B
	wait_send_A_complete
	wait_send_B_complete
```
The remote machine might receive B first then A, and do Operate_on_A on B and do Operate_on_B on A. 

Furthermore, the scalability could be poor, since each threadblock need a QP/CQ and when the number of GPU increase, this number could become very large.

## MVAPICH2
For MVAPICH2, the data movement is between GPU and NIC but the control path is on CPU. As explained in the section GPUrdma

## GPUDirect Async Stream Asynchronous Model
Comparing with GPUrdma, the data path is still directly from GPU and NIC. However, the control path is little different. On GPUrdma, all the control path is on GPU, but in GPUDirect Async, setting up the information for communication like create QP, insert a WQE is called on CPU, ring the doorbell and polling the CQE of corresponding request is on GPU. So in GPUDirect Async, GPU and CPU colleborate and CPU gives all the computation and communication instructions then walk away, GPU takes those instructions, launch computation kernels, ring the doorbell and poll on the CQ. This process can be shown as below:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/SA.jpg)
So the memory mapping also change a little, now QP and CQ are all on the CPU memory and GPU will poll on CPU memory and ring the doorbell on HCA.
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/SAGPUCPUHCA.jpg)
Now, having the doorbell register and CQs mapped in the GPU, it is possible to access them using CUDA kernel threads (Kernel-Initiated communication model shown later) or we can instruct CUDA stream to poll or ring the doorbell using CUDA driver functions which we name this Stream Asynchronouse Model (SA).

So for SA model, when the communication happens need to be known in advance and also the size of transfer and location. Computation kernels need to end in order to launch communication kernel. The communication kernel and computation kernel are synchronouse within same stream and streams are asynchronouse with respect to each other. This approach is easy to use and modify the original code.

## GPUDirect Async Kernel-Initiated Communication Model
The underlying mapping the control path is the same as the GPUDirect Async Stream Asynchronouse Model, they differ from the way they use their ability to ring the doorbell and poll on the CQ. So in this KI model, computation and communication can be fused using kernel fusion technique. As in the SA model, the CPU prepares the communication descriptors and later those communications are triggered directly by threads in the CUDA kernels KI instead of being triggered by the CUDA streams. Say the task is:
```bash
	Receive data A
	Do computation task C on data A
	Do computation task A on data C
	Send data C
	Do computation task B on data B
```
The dependency is computation task C should be on A, Send data C should be after finishing computation task A on C. Data B and computation B are irrelevant.
KI communication model is very task parallelism model. For the above task, we will have at least N+M+1 blocks, where N is the number of blocks required to compute type A tasks before the send operation, M is the number of blocks required by type C tasks working on received data plus 1 block, used to poll the CQs as shown in below figure:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/master/images/KI.jpg)
In this KI model, the kernel fusion technique is combined with a dynamic dispatcher which uses atomic operations to pick each thread block and to assign it to the right task follow some rules like the receive operation must not prevent the send operation from starting.

Since in this model, a CUDA thread use simple value assignments and comparisons inside the code to ring the doorbell or poll the CQs, it won't take as many resourse as GPUrdma.

## Comparison
So the most widely use model is that the data transfer calls are invoked by the CPU, while ensuring that the GPU kernel that generates the data terminates before the tranfer start by synchronizing. The problems with this scheme are that:
1. The codes are forced to be programmed in a bulk-synchronouse way, then makes overlapping computation and communications challenging. 
2. Synchronizing between kernel execution and network operations is complicated since GPUs and NICs expose different synchronization mechanisms and event interfaces, and even more challenging when executing multiple kernels and I/O calls in parallel to achieve pipelinging.
3. Since to tranfer data, the kernel need to end. If each kernel runs long enough, it may justify the kernel launch overhead. But if the kernel is small then the kernel launch overhead can be significant. Also since we have to end the kernel to communicate, we lose many opportunities to use shared memory to achieve better performance.
4. The messages to be transferred to other machines must be accumulated in internal GPU buffers during the kernel execution. This in turn increase the kernel memory consumption or constrains the amount of data kernel may process each time.
5. GPUrdma enable a more natural application design in which computations and I/O are interleaved, and the I/O challs are performed directly from GPU code. As a result, rather than terminate the kernel for performing I/O, we can execute long running threadblocks throughout the entire lifetime of the application in a persistent kernel style.
6. GPUrdma takes non-trivial resource from GPU and could affect occupancy.
7. GPUDirect Async SA model may not allow for a persistent thread scheme, but it is also simple to program.

## One-sided operation VS two-sided operation
Point-to-point operations of the two-sided type: they require the co-operation of a sender and receiver. This co-operation could be loose: you can post a receive with MPI\_ANY\_SOURCE as sender, but there had to be both a send and receive call. 

In one-sided MPI operations, also known as RDMA or RMA operations, there are still two processes involved: the origin , which is the process that originates the transfer, whether this is a `put' or a `get', and the target whose memory is being accessed. Unlike with two-sided operations, the target does not perform an action that is the counterpart of the action on the origin.

That does not mean that the origin can access arbitrary data on the target at arbitrary times. First of all, one-sided communication in MPI is limited to accessing only a specifically declared memory area on the target: the target declares an area of user-space memory that is accessible to other processes. This is known as a window . Windows limit how origin processes can access the target's memory: you can only `get' data from a window or `put' it into a window; all the other memory is not reachable from other processes.

The alternative to having windows is to use distributed shared memory or virtual shared memory : memory is distributed but acts as if it shared. The so-called PGAS languages such as UPC use this model. The MPI RMA model makes it possible to lock a window which makes programming slightly more cumbersome, but the implementation more efficient.

Within one-sided communication, MPI has two modes: active RMA and passive RMA. In active RMA , or active target synchronization , the target sets boundaries on the time period (the `epoch') during which its window can be accessed. The main advantage of this mode is that the origin program can perform many small transfers, which are aggregated behind the scenes. Active RMA acts much like asynchronous transfer with a concluding Waitall .

In passive RMA , or passive target synchronization , the target process puts no limitation on when its window can be accessed. ( PGAS languages such as UPC are based on this model: data is simply read or written at will.) While intuitively it is attractive to be able to write to and read from a target at arbitrary time, there are problems. For instance, it requires a remote agent on the target, which may interfere with execution of the main thread, or conversely it may not be activated at the optimal time. Passive RMA is also very hard to debug and can lead to strange deadlocks.

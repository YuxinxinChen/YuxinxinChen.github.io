## GPU Memory Bank 

In modern GPU architectures, shared memory plays a crucial role in enabling efficient parallel processing. This high-performance, on-chip memory allows threads within a thread block to share data and communicate with each other quickly. Shared memory is organized into a set of memory banks, which allows multiple threads to access memory concurrently. However, understanding and managing bank conflicts is essential to optimize the performance of GPU programs. In this tutorial, we will explore the fundamental concepts of GPU memory banks in shared memory, their importance in parallel processing, and techniques for minimizing bank conflicts to harness the full potential of shared memory.

## Visionlization of Memory Bank
Take the memory bank in Volta GPUs as an illustrative example.
In Volta 100, there are 32 banks, each with 4 bytes wide. Therefore:
- Bandwidth: 4 bytes per bank per clock per SM, thus 128 bytes per clock per SM
- V100: ~14 TB/s aggregate across 80 SMs

Mapping addresses to banks:
- Successive 4-bytes words go to successive banks
- Bank index computation examples:
  - (4B word index) % 32
  - ((1B word index)/4) % 32
  - 8B word spans two successive banks

Logical View of SMEM Banks:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/new_home_page/images/membank.png)


## Bank Conflict
A bank conflict occurs when, inside a **warp**, 2 ore more threads access **different** 4B words within the **same** bank. As the results, the access (load and store) to the same bank will be serialized, increasing access latency. In the worst case, all 32 threads within the warp access different 4B words in the same bank, resulting 32-way conflict and 32 serialized thread access.

There is **no** bank conflict if:
- Several threads access the same 4-byte word
- Several threads access differnt bytes of the same 4-byte word
- Several threads access different banks

## Lab Section: Memory bank Conflict
### First Experiment
We could test the memory bank conflict by access the shared memory with different stride and measure the time:
```
template<int N>
__global__ void test1(int stride, float *out)
{
    long long int start = 0, stop = 0;
    double time, usec;
    __shared__ float shareM[N];
    for(int i=threadIdx.x; i<N; i+=blockDim.x)
        shareM[i] = i;

    __syncthreads();
    start = clock64();

    for(int i=0; i<10; i++)
    {
        float item = shareM[(threadIdx.x*stride)];
        item++;
        shareM[(threadIdx.x*stride)] = item;
        __syncthreads();
    }

    stop = clock64();
    if(!threadIdx.x && !blockIdx.x) {
        time = (stop-start);
        usec = time*1000/clockrate;
        //printf("time in usec: %.2f \n", usec);
        printf("%.2f\n", usec);
    }

    for(int i=threadIdx.x; i<N; i+=blockDim.x)
        out[i] = shareM[i];
}
```
We ran the above code with different stride and measure the time:
```
for(int stride = 64; stride > 0; stride--) {
    test1<N><<<1, 32>>>(stride, out);
    CUDA_CHECK(cudaDeviceSynchronize());
}
```
If we plot the time on y-axis, stride size on x-axis, we get this plot:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/new_home_page/images/stride_time.png)

To understand what is going on, let's visualize several cases:
When stride is 32, we got the one of the worst performance. If we visualize it:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/new_home_page/images/stride_32.png)
As shown in the above visualization, all 32 threads access the same bank, resulting 32 serialized access

For stride 16, we visualize it:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/new_home_page/images/stride_16.png)
As shown in the above visualization, 16 threads access the bank 0, the other 16 threads access the bank 16, resulting 16 serialized access time step

For stride 5, we visualize it:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/new_home_page/images/stride_5.png)
As shown in the above visualization, all 32 threads access different banks, thus no bank conflict 

For stride 1, we visualize it:
![](https://github.com/YuxinxinChen/YuxinxinChen.github.io/blob/new_home_page/images/stride_1.png)
As shown in the above visualization, all 32 threads access different banks, thus no bank conflict 


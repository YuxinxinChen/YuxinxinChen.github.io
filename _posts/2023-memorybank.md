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

## Lab Section: Shared Memory Allocation Pattern
There are two ways of allocating shared memory:
- Static allocation by `__shared__ int a[3]` where the size of shared memory is a static value
- Dynamic allocation by `extern __shared__ int a[]` where the sizeof of dynamic shared memory is passed from kernel invokation by bytes.

To study the allocation pattern when both static and dynamic allocation are used, we run following code with differnt `N`:
```
template<int N>
__global__ void test2()
{
    __shared__ float sharedM1[1];
    __shared__ float sharedM2[1];

    extern __shared__ float externSpace[];

    __shared__ float sharedM3[N];

    if(threadIdx.x == 0)
    {
        printf("sharedM1 %p\n", sharedM1);
        printf("sharedM2 %p\n", sharedM2);
        printf("externSpace %p\n", externSpace);
        printf("sharedM3 %p\n", sharedM3);
        int n = (externSpace-sharedM1);
        printf("extern-sharedM1 %d\n", n);
        printf("static shared: %d\n", 1+1+N);
    }
}
```
We run the code with different `N`:
```
test2<1><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 1);
    test2<2><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 2);
    test2<3><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 3);
    test2<4><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 4);
    test2<5><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 5);
    test2<6><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 6);
    test2<7><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 7);
    test2<8><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 8);
    test2<9><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 9);
    test2<10><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 10);
    test2<11><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 11);
    test2<12><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 12);
    test2<13><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 13);
    test2<14><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 14);
    test2<15><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 15);
    test2<16><<<1,32, 7*sizeof(float)>>>();
    CUDA_CHECK(cudaDeviceSynchronize());
    printf("----------N: %d-------------\n", 16);
``` 

We got the results:
```
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000010
sharedM3 0x7f1a80000008
extern-sharedM1 4
static shared: 3
----------N: 1-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000010
sharedM3 0x7f1a80000008
extern-sharedM1 4
static shared: 4
----------N: 2-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000020
sharedM3 0x7f1a80000008
extern-sharedM1 8
static shared: 5
----------N: 3-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000020
sharedM3 0x7f1a80000008
extern-sharedM1 8
static shared: 6
----------N: 4-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000020
sharedM3 0x7f1a80000008
extern-sharedM1 8
static shared: 7
----------N: 5-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000020
sharedM3 0x7f1a80000008
extern-sharedM1 8
static shared: 8
----------N: 6-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000030
sharedM3 0x7f1a80000008
extern-sharedM1 12
static shared: 9
----------N: 7-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000030
sharedM3 0x7f1a80000008
extern-sharedM1 12
static shared: 10
----------N: 8-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000030
sharedM3 0x7f1a80000008
extern-sharedM1 12
static shared: 11
----------N: 9-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000030
sharedM3 0x7f1a80000008
extern-sharedM1 12
static shared: 12
----------N: 10-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000040
sharedM3 0x7f1a80000008
extern-sharedM1 16
static shared: 13
----------N: 11-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000040
sharedM3 0x7f1a80000008
extern-sharedM1 16
static shared: 14
----------N: 12-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000040
sharedM3 0x7f1a80000008
extern-sharedM1 16
static shared: 15
----------N: 13-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000040
sharedM3 0x7f1a80000008
extern-sharedM1 16
static shared: 16
----------N: 14-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000050
sharedM3 0x7f1a80000008
extern-sharedM1 20
static shared: 17
----------N: 15-------------
sharedM1 0x7f1a80000000
sharedM2 0x7f1a80000004
externSpace 0x7f1a80000050
sharedM3 0x7f1a80000008
extern-sharedM1 20
static shared: 18
----------N: 16-------------
```

The first observation is that regardless of where static shared memory allocation is called, the static shared memory are allocated aggregately in a big chunk. There is no padding in between and allocated continuesly. The static shared memory are always allocated before dynamic shared memory.

The second abservation is that dynamic shared memory are allocated in multiple of 16 bytes. 

Based on this pattern, it is easy for us to reason about shared memory bank conflict when multiple types of shared memory are used.

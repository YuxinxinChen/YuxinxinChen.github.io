---
layout: post
title: "Gunrock Reading Notes"
data: 2017-10-10
tags: [reading notes, GPU]
comments: true
share: false
---

## Connected Components

Usually, there are two ways to find the connected components: label propagation and hook and pointer jumping (SV). 
Label propagation: 

```c
while there is at least one edge (u,v) such that label(u)!=label(v)
	for each edge (i,j) such that label(i)!=label(j)
		min = Min(label(i), lable(j))
		label(i) = min
		label(j) = min
	end for
end while	
```
Let x be the smallest label, and let v be the node whose initial label is x. The label x will propagate from v to the nodes at distance 1 from v, then to the nodes at distance 2 from v... and eventually to the farthest vertices from v, like a wave. In the worst case, n nodes form a path with the minimum label on one end, the runtime will be O(n). But in practice one would not be so unlucky.

Hook and jumping:

Oldest version: Shiloach & Vishkin
```c
for each node u do in parallel
	Parent(u) = u
end for
while trees are not star-trees or there are edges between star-trees
	for each edge (u,v) where u or v is a root or a level-1 node do in parallel
		hook(u,v)
	end for
	for each node u do in parallel
		single-pointer-jumping(u)
	end for
	for each stagnant edge (u,v) do in parallel
		hook_stagnant(u,v) anyway
	end for
	for each node u do in parallel
		single-pointer-jumping(u)
	end for	
end while
```

```c
hook(u,v)
if u is a root or a level-1 node
	if Parent(u) > Parent(v)
		Parent(Parent(u)) = Parent(v)
	end if
end if
if v is a root or a level-1 node
	if Parent(v) > Parent(u)
		Parent(Parent(v)) = Parent(u)
	end if
end if		  
```

```c
single-point-jumping(u)
Parent(u) = Parent(Parent(u))
```

```c
hook(u,v)
if u is a root or a level-1 node
	Parent(Parent(u)) = Parent(v)
end if

if v is a root or a level-1 node
	Parent(Parent(v)) = Parent(u)
end if		  
```

A node u is a root if Parent(u) = u. A node u is a level-1 node if Parent(Parent(u)) = Parent(u). There are cases when hooks fails because the root or level-1 nodes’s parent is smaller than the parent of the other side of the edge, such an edge is called a stagnant edge. Then after all the available hook are finished, we call hook-stagnant to hook the stagnant edges anyway.
The SV algorithm terminates after O(log(|V|)) iterations of the while loop. Each iteration has a constant cost, so the runtime is O(log(|V|)).

Most commonly used version: Soman
```c
while trees are not star-trees or there are edges between star-trees
	for each edge (u,v) do in parallel
		hook(u,v)
	end for
	for each node u do in parallel
		multi-pointer-jumping(u)
	end for
end while
```

```c
hook(u,v)
if Parent(u) > Parent(v)
	Parent(u) = Parent(v)
else Parent(v) = Parent(u)
end if
```

```c
multi-pointer-jumping(u)
while Parent(Parent(u))!=Parent(u)
	Parent(u) = Parent(Parent(u))
end while
```

Similar to SV, Soman’s algorithm alternates between parallel hooking and parallel multi-pointer-jumping. The main difference is that instead of single-pointer-jumping used in SV, Soman performs multi-pointer jumping, i.e. we do iterative pointer jumping till every tree is a star-tree. Thus, each parallel hooking is performed on depth-1 trees. This exposes more candidate edges for hooking. In contrast, in SV, some edges have both endpoints deeper than level 1, and are ineligible for hooking.
Furthermore, if we flatten the tree sooner, the next hooking has a better chance to hook to a root. SV may create a very deep tree. 

However, in Soman, in the worst case, the while loop will need O(log(n)) iterations and each multi-pointer-jumping will have O(log(n)) iterations, so the runtime is O(log(n)^2). It seems that Soman has a worse runtime, but it is also favored by GPU since multi-pointer-jumping can be written into one kernel and has better memory access coalesce and less branch divergence. Think about in this way: we aggregate all the possible pointer jumping together, then each memory fetch (several mem cache lines) will get better utilized in terms of how much pointer jumping can do comparing to SV version.  At the same time, because it need to jump more than once, the ratio of runtime of the two divergent threads is large. Hence thread serialization penalty was minimized.

Note that the parallel hooks are not locked, so some hooking may end up being overwritten. This race condition is benign, as it only affects the runtime of the algorithm, but not the final correctness of the output CC.

Adaptive CC:
```c
for each node u do in parallel
	Parent(u) = u
end for
s = 2|E|/|V| 
let E_1 ….E_s be s distinct subsets of E
for each E_i
	for each edge (u,v) in E_i do in parallel
		AtomicHook((u,v))
	end for
	for all v in V do in parallel
		MultiJump(v)
	end for
end for
```

```c
AtomicHook((u,v)):
while Parent(u)!=Parent(v) do
	H = max{Parent(u), Parent(v)}
	L = min{Parent(u), Parent(v)}
	lock Parent(H)
		if Parent(H) = H: then
			Parent(H) = L
			return
		else
			u = Parent(H)
			v = L
		end if
	end lock
end while
```
In adaptive CC, when hooking an edge, the node with a higher Parent ID will traverse all the way to the root and the Parent ID assignments are atomic, then after AtomicHook has been called once for each edge (in parallel), the spanning tree or spanning forest is formed. After calling MultiJump on each node, every spanning tree will be contracted into a star tree. Parent IDs are updated to the root IDs of the trees, which are also the connected component IDs. However, with only one AtomicHook followed by a contraction, it may eliminates the ability to perform constant Contraction operations between each Hook round compared to Soman. Think about it this way: if we flatten the tree in advance, it takes shorter path to traverse to the root when doing AtomicHook. So in adaptive CC, one divides the edge-list into approximately $2|E|/|V|$ (i.e. average node degree) disjoint segments. Then for each segment, only |V| Hook operations are always performed, which is proportional to the |V| memory workspace. This reduces atomic contention compared to the case when we do not use segments (i.e. |E| hook operations are performed to the |V| memory workspace). We loop over the segments E_1…E_s sequentially. For each segment E_i, we perform AtomicHoook on all edges in E_i in parallel, followed by MultiJump on all nodes in parallel. MultiJump flattens all spanning trees and minimizes atomic operations for the next segment Hook. As average degree increases, more MultiJump operations will be performed, but their O(|V|) cost becomes minor compared to the dominating O(|E|) complexity of the overall algorithm. 

The good thing about Grout adaptive CC is that since it only requires 2\*number of segments global barrier, it keeps global barrier minimized. Instead it uses atomic operations to prevent overwritten and thus needs less global barriers. #atomic operations are good friends to asynch.

Some algorithm which claims it is better than adaptive CC:
```c
ECL_CC(V,E):
Init(V, nstat)
Compute(V,E,nstat)
Flatten(V,nstat)
```

```c
Init(V, nstat)
nstat = {0,...., |V|-1}
for each vertex v in V do in parallel:
	nstat[v] = First neighbor smaller than v
end for
```

```c
Compute(V,E,nstat)
for each v in V do in parallel:
	vstat = representative(v,nstat)
	for each edge(u,v) in E do in parallel:
		if(v>u)
			ostat = representative(u, nstat)
			if(vstat < ostat)
				nstat[ostat] = vstat
			else 
				nstat[vstat] = ostat
			end if
		end if
	end for
end for
```

```c
Flatten(V,nstat)
for each vertex v in V do in parallel:
	vstat = nstat[v]
	while(vstat > nstat[vstat])
		vstat = nstat[vstat]
	end while
end for
```

```c
Representative(v,nstat)
curr = nstat[v]
if(curr!=v)
	prev = v
	next = nstat[curr]
	while(curr > next)
		nstat[prev] = next
		prev = curr
		curr = next
	end while
end if
```

ECL-CC has several differences comparing to adaptive CC: 1) Instead of initializing the Parent ID of each vertex with its own iD, it initializes the Parent ID with the first neighbor’s ID smaller than it. 2) when hooking, it still traces back to the root which is the same as adaptive CC and use atomic operation to prevent overwritten, but instead of root tracing only one side of a edge as in the adaptive CC, it traces the root for vertices on both ends of the edge. 3) As it traces back to the root, it also conducts immediate pointer jumping along the way. In addition, it tries to do load balancing by grouping the nodes by their degrees. It applies different scope of thread to different degree vertices: thread-level to vertices with degree less than 16 and assign a vertex to each thread, warp-level to vertices with degree between 16 to 512 and assign a warp to each vertex, block-level to vertices with degree larger than 512 and assign a block to each vertex. Hmmm....it claims it has better load balance. So why not just assign each thread to each edge but use load balancing scheme to assign different threads to each vertex? One benefit could be in ECL-CC, it tries to find the root of vertex u and its neighbors, we could find u's root only once for all its neighbors. Then why we want to find roots of both end node of a edge? We might think connecting two trees by the root should give us a shallower tree thus saving some MultiJump operation. But creating the vertex degree list is not free. I am not sure, in its experiment, if it includes the time to create the vertex degree double side list and its results are only slightly better than Grout adaptive CC.

From above exhibition of CC algorithms, we may have a feeling that at the beginning, to favor a parallel execution, CC algorithm relaxes the data correct requirement by allowing data overwriting and doing more iterations instead. However those iterations requires global barriers in between. Then when we go towards asynchronous approach because of expensive global barrier, we return back to the original approach: using atomic operation to ensure the data correctness and disallow overwriting, so reducing number of iterations (one iteration is one global barrier) we do. So I would say that the choice of the algorithm depends on which is the lesser of two evils between atomic operations and global barriers.

Unfortunately, the memory access pattern of SV, Soman, Adaptive CC and ECL-CC are unfavorable for implementation on a distributed memory system. The “Jump” step accesses the grandparent of a vertex u stored as D[D[u]], where D represents the parent relationship. When D is distributed among processors, accessing D[D[u]] generates erratic remote access. In addition, there will be a flood of messages going to the processor that owns the root of the tree as the root is accessed by each "pointer jumping". In addition, for both BFS and SV, the use of global barriers causes scaling problem when many processors are available. SV takes O(log n) barriers, while parallel BFS needs O(d) barriers. 

There are some other asynchronous algorithms which use a very different approach from hooking+pointer-jumping combined with atomic operations. It uses DFS to construct spanning trees. We call it Cong

In Cong, it has two assumptions: global barrier and communication are expensive which is true for distributed system. So it aims to reduce both of them so it adopts DFS.
```c
DFS_T(u)
if u is owned by local processor
	for each neighbor v of u
		send tuple (u,v) to the processor that owns v
	end for
end if
wait for termination
```
```c
Msg_Handler((u,v))
if v is not visited
	set v as visited
	set v's parent as u
	for each neighbor t of v
		send tuple (v, t) to the processor that owns t
	end for
end if
```
 
In the above code, visited list is maintained using atomic operation and the termination condition for each machine is all vertices assigned on that machine have been visited (which do not require to send back messages). Then the global barrier is replaced by recursive call to the next level of nodes which can be dynamically triggered. While you may argue the adaptive CC algorithm also minimizes global barriers, but it needs to travers back to the root for each edge, this traversal may result in tons of messages especially when the paths to roots are long. Then since each edge will trigger a traversal of length at least one, then there will be at least |E| messages flying around. However, because in Cong's method, each edge will only trigger a DFS on the other side of the edge and the other side of the edge may reside in the same machine, then there is at most |E| messages in the network. However, DFS has some other problems. Compared to adaptive CC, it requires O(|V|) runtime in the worst case since it exposes less parallelism, while adaptive CC is O(log|V|) if communication is ideal. Also DFS_T which dynamically forks threads is hard to map onto GPU.



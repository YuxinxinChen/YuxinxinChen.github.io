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
In the above AtomicHook code, it just travese all the way to the root and the overriden is atomic, then after one atomicHook with repected to all edges, the spanning tree or spanning forest are formed. After doing a multiJump on spanninging tree or forest, CC ids are updated. However, with only one atomicHook followed by a contraction, it may eliminates the ability to perform constant Contraction operations between each Hook round. Or think in this way, if we flatten the tree in advance, it takes shorter path to travers to the root when do atomicHook. So in the adaptive CC, it divide the edge-list into several segments which is approximatly $2|E|/|V|$. s is number of segments and also approximately average degree. Then Hook operations are always performed over segments proportional to the |V| memory workspace, reducing atomic contention. Each such segment Hook, is followd by a Contraction operation, flattening all component trees and minimizing atomic operations for the next segment Hook. As average degree increases, more Contraction operations will be performed, but their O(|V|) cost becomes minor compared to the dominating O(|E|) complexity of the overall algorithm. 

The good thing about Grout adaptive CC is that since it only requires 2\*number of segments iterations, it keeps global barrier minimized. Instead it uses more atomic operation to prevent overwritten then needs less global barrierr. #atomic operations are good friends to asynch.

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
	nstat[v] <-- First neighbor smaller than v
end for
```

```c
Compute(V,E,nstat)
for each v in V do in parallel:
	vstat <-- representative(v,nstat)
	for each edge(u,v) in E do in parallel:
		if(v>u)
			ostat <-- representative(u, nstat)
			if(vstat < ostat)
				nstat[ostat] <-- vstat
			else 
				nstat[vstat] <-- ostat
			end if
		end if
	end for
end for
```

```c
Flatten(V,nstat)
for each vertex v in V do in parallel:
	vstat <-- nstat[v]
	while(vstat > nstat[vstat])
		vstat <-- nstat[vstat]
	end while
end for
```

```c
Representative(v,nstat)
curr <-- nstat[v]
if(curr!=v)
	prev <-- v
	next <-- nstat[curr]
	while(curr > next)
		nstat[prev] <-- next
		prev <-- curr
		curr <-- next
	end while
end if
```

The above algorithm has several difference comparing the adaptive CC: 1) Instead of initilize the root of each vertex with its own id, it initilizes the root with the first neighbor smaller than it. 2) when hooking, it still trace back to the root same adaptive CC and use atomic operation to prevent overwitten, but instead of tracing only one as in the adaptive CC, it traces back the vertices on both end of edge. 3) When it trace back to the root, it also conducts an immediate pointer jumping along the way. In addition, it tries to do load balancing by dividing the vertex by its degree. It apply different scopy of thread to different degree vertex: thread-level to vertex with degree less than 16 and assigne a vertex to each thread, warp-level to vertex with degree between 16 to 512 and assign a warp to a vertex, block-level to vertex with degree larger than 512 and assign each block to a vertex. Hmmm....it claims it has better load balance. So why not just assign each thread to each edge but to do that in vertex way? One benefit could be in ECL-CC, we try to find the root of both side of an edge, for vertex u, we could find u's root only once and we might think connect two by the root should give us a shallower tree thus saving some contraction operation. But creating the vertex degree list is not free. I am not sure, in its experiment, if it includes the time to create the vertex degree double side list and it only shows slightly better than Grout adaptive CC.

From above background algorithm, we may have a feeling, that at the beginning, to favor a parallel execution, CC algorithm relaxes the data correct requirment to do more number of iterations allowing data overwritten. However those iterations requires global barrier between. Then when we go toward asynchronous approach because of expensive global barrier, we return to the original approach: using atomic operation to ensure the data correctness and disallow overwritten, then reducing number of iterations we do. So I would say that the choice of the algorithm depends on which is the lesser of two evils between atomic operations and global barriers.

Unfortunately, the memory access pattern of SV, Soman, Adaptive CC and ECL-CC are unfovorable for implementation on a distributed memory system. The "shortcutting" step accesses the grandparent of a vertex u stored as D[D[u]], where D represents the parent relationship. When D is distributed among processors, accessing D[D[u]] generates erratic remote access. In addition, there will be a flood of messages going to the processor that owns the root of the tree as the root is accessed by each "pointer jumping". In addition, for both BFS and SV, the use of global barriers caues scaling problem when many processors are available. SV takes O(log n) barriers, while parallel BFS needs O(d) barriers. 

There are some other asynchronous algorithms use a very different appoarch instead of hooking+pointer jumping combined with atomic operations. It uses DFS to construct spanning tree. We call it Cong

In Cong, it has two assumptions: global barrier and communication are expensive which is true for distributed system. So it aims to reduce both of them so it adopt DFS.
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
 
In the above code, visited list is matained using atomic operation and the termination is all vertices assigned on that machine have been visited (which do not require to send back messages). Then the global barrier is replaced by recursive call to the next level of nodes which can be dynamically triggered. While you may argue the adaptive CC also minimized global barriers but it needs to travers back to root for each edge, this travers may result ton of message if there are many long path. Then since each edge will trigger a travers at least length of one, then there will be at least |E| messages flying around. However, because in Cong's method, each edge will only trigger a DFS on the other side of the edge and the other side of the edge may reside in the same machine, then there is almost |E| messages in the network. However, DFS has some other problems. Comparing the adaptive CC, it requires O(|V|) runtime in the worst case since it exposures less parallel, while adaptive CC is O(log|V|) if communication is ideal. Also DFS which dynamically fork threads is hard to map onto GPU.



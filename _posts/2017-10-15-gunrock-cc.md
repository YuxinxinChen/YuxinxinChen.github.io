---
layout: post
title: "Gunrock Reading Notes"
data: 2017-10-10
tags: [reading notes, GPU]
comments: true
share: false
---

## Connected Components

Usually, there are two ways to find the connected components: label propagation and hook and pointer jumping (SV). TODO label propogation and hook and pointer jumping
Label propagation: 

```c
while(there is at least one edge (u,v), Parent(u)!=Parent(v)):
	for edges whose parent are not equal (i,j):
		Max(Parent(i), Parent(j)) = Min(Parent(i), Parent(j))
	end for
end while	
```
Then the Parent label will propagate from the vertex with min id to the vertex whose distance is 1 from min-id vertex, then distance 2 ....., farest vertex away from min-id vertex, like a wave. In the worst case, n nodes form a path with min-id in either of end, the runtime will be O(n). The modification of the label still need a lock since it may receive the propagation from all its neighbors. 

Hook and jumping:

Oldest version: Shiloach & Vishkin
```c
while(at least one edge is not hookded and trees are not star-trees):
	for all edges try 1-level hook do in parallel
	for each tree try single pointer jumping do in parallel
	if stagnent happens:
		hook anyway do in parallel
		single pointer jumping do in parallel
end while
```
In above pseudocode code, 1-level hook means if there is a edge (u,v), Parent(u) > Parent(v) and u is root or has distance 1 from its root, then we hook the tree of u onto Parent of v. Single pointer jumping is, within a tree, for each node, do Parent(node) = Parent(Parent(node)). Stagnent happens when the level-1 node of any of the trees has a edge out but they couldn't hook up together, at the same time, every tree's root can not hook up onto other place or be hooked up by other nodes. Then we know we should been about to hook the level-1 node' root onto other tree but we couldn't because of its parent id is larger, so we choose to hook it up anyway, excluding the situation that hooking process is serialized because of small-id root are connected via a large-id root and making the hooking process always has a small constant cost. Then the runtime of SV algorithm is mainly occupied by single point jumping which has runtime of O(log(d)) where d is the depth of spinning tree formed. In the worst case, d can be n, 1 in the best case and log(n) in general case. So in the worst case, the runtime is O(log(log(n))) but generally there is not much difference between log(n) and log(log(n)). 

Used most version: Soman
```c
while(at least one edge is not hooked):
	for all edges do in parallel:
		hook
	end for
	for each tree do in parallel:
		multi pointer jumping
	end for
end while
```
In above version, the algorithmic complexity is the same as SV version, the sequence we do hooking and pointer jumping doesn't change algorithmic complexity. In stead of doing a single pointer jumping after a parallel hooking, we do a iterative pointer jumping till that tree becomes a star tree. Then the hooking after multi pointer jumping is processed on a level-1 tree. So we exposure more hooking candidate nodes for the next hooking since in SV, the candidate nodes are only level-1 nodes and roots. Furthermore, if we flatten the tree more soon, the next hooking has a better chane to hook to a root. SV hooking may create a very deep tree. However, in the worst case, the while will need O(log(n)) iterations and each multi pointer jumping will have O(log(n)) iterations, then the runtime is O(log(n)^2). Seems Soman has a worse runtime, but it is also favored by GPU since multi pointer jumping can be written into one kernel and has better memory access coalease and less branch divergence. Think about in this way, we aggregate all the possible pointer jumping together, then each memory fetch (several mem cache lines) will get better utilized in terms of how much pointer jumping can do comparing to SV version. At the same time, because it need to jump more than once, the ratio of runtime of the two divergent threads is large. Hence thread serialization penalty was minimized. 

Adaptive CC:
```math
let \pi: \pi(vertex) <-- vertex
let s <-- 2|E|/|V| 
let {E_i}^{s}_{i=1} be s distinct subsets of E
for i <-- 1, s do
	for all e in E_i do in parallel
		AtomicHook(e,\pi)
	end for
	for all v in V do in parallel
		MultiJump(v, \pi)
	end for
end for
return \pi
```

```math
AtomicHook(e, \pi):
while \pi(u)!=\pi(v) do
	H <-- max{\pi(u), \pi(v)}
	L <-- min{\pi(u), \pi(v)}
	lock \pi(H)
		if \pi(H) = H: then
			\pi(H) <-- L
			return
		else
			u <-- \pi(H)
			v <-- L
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



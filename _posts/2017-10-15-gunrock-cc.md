---
layout: post
title: "Connected Components Reading Notes"
data: 2017-10-15
tags: [reading notes, GPU]
comments: true
share: false
---

## Weakly Connected Components

Usually, there are two ways to find the connected components: label propagation and hook and pointer jumping (SV). 

### Label propagation: 

```c
while there is at least one edge (u,v) such that label(u)!=label(v)
	for each edge (i,j) such that label(i)!=label(j)
		min = Min(label(i), lable(j))
		label(i) = min
		label(j) = min
	end for
end while	
```


Hook and jumping:

### Oldest version: Shiloach & Vishkin

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

### Most commonly used version: Soman

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

### Adaptive CC:

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

### Some algorithm which claims it is better than adaptive CC:

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

### Cong's algorithm

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

## Pull-based model VS Push-based model

Pull-based model: When data exchanges needed between two or more threads, processor, machines, in pull-based approach, one thread will try to access the data another threads is working on (processor access data another processor working on, or one machine send an mem access request on network to another machine). Because it might pull inconsistent data from any concurrently executing neighbours, pull-based asynchronous implementation need to use lock to ensure data inconsistency.Gather, Apply and Scatter (GAS) model is pull-based and its asynchronous implementation uses distributed locking to ensure data consistency.

Push-based model: The information is sended explicitly after finishing the computation related to that piece of information. Then the receiver receive the consistent information passively. However it requires the sender push the message explicitly. Since messages are buffered in a local message store, concurrent reads and writes to the store can be handled locally with local locks or lock-free data structure. 

## Strongly Connected Components (SCC)

### Serial Algorithm

Tarjan's algorithm is a serial algorithm using DFS to find the SCC. The algorithm perform linear O(|V|+|E|) work. The reason the serial algorithm is mentioned here because sometimes, using serial algorithm is faster = =

### Forward-Backward Algorithm

```c
FW_BW(V)
if V is not empty then
	return
end if
Select a pivot u in V
D = BFS(G(V, E(V)), u)
P = BFS(G(V, E'(V)), u)
R = (V exclude (P union D))
S = P intersection D

launch do in parallel
	FW_BW(D exclude S)	
	FW_BW(P exclude S)
	FW_BW(R)
end do
```
In FW_BW pseudo code, V denotes the set of nodes in the graph, E(V) is the set of outgoing edges for set nodes V, and E'(V) is the set of incoming edges  for set nodes V. Given the graph G(V, E), a pivot vertex u is selected. This can be done either randomly or through simple heuristics. A BFS search is conducted starting from the vertex to determine all nodes which are reachable from u (the forward sweep). These nodes form the descendant set D. Another BFS is performed from u, but on G(V,E'). This search (the backward sweep) will find the set P of all nodes that can reach u, called the predecessor set. The intersection of these two sets forms an SCC that has the pivot u in it. If we remove all nodes in S from the graph, we can have up to three remainning disjoint node set: (D exclude S), (P exclude S) and the remainder R, which is the set of nodes that we have not explored during either search from u. The FW_BW algorithm can then be recursively called on each of these three sets. Note, there is parallelism on two levels. As the three sets are disjoint, they can each be explored in parallel. Also, we do not require any vertex ordering within each set, just reachability. Therefore, each of the forward and backward searches can be easily parallelized or run concurrently. For graphs with bounded constant node degree, FW_BW is shown to perform O(nlog(n)) expected case work.	

Usually, a rountine called trimming is performed before executing FW-BW. The procedure is quite simple: all nodes that have an in-degree or out-degree of zero are removed. Trimming can also be performed recursively, as removing a node will change the effective degrees of its neighbors. Single iteration of trimming is called simple trimming and iterative trimming is called complete trimming. Usually, the marginal benefit to do more triming is decreasing.

### Coloring Algorithm

```c
ColorSCC(G(V,E))
while G is not empty do
	for all u in V do in parallel
		Colors(u) = u
	end for
	while at least one node has changed colors do
		for all u in V do in parallel
			for all (u,v) in E do in parallel
				if Colors(u) > Colors(v)
					Colors(v) = Colors(u)
				end if
			end for
		end for
	end while

	for all unique c in Colors in parallel do
		Vc = {u in V: Colors(u) = c}
		SCVc = BFS(G(Vc, E'(Vc)), u)
		V = (V excludes SCVc)
	end for
end while
```

Assume that the graph nodes are numbered from 1 to n. The algorithm starts by initializing elements of array Colors to node ID. The values are then propagated outward from each node in the graph, until there are no further changes to Colors. Then effectively partitions the graph into disjoint set. As we initialized Colors to node ID, there is a unique node correponding to every distinct c in Colors. We consider u = c as the root of a new SCC, SCVc. The set of reachable nodes in the backward seep from u of nodes of the same color(Vc) belong to this SCVc. We then remove all these nodes from V and proceed to the next color/iteration. The two subroutines ameable to parallelization are the color propagation step and the backward sweep. In a graph with a very large SCC and high diameter, the color of the root node has to be propagated to all of the nodes in the SCC, limiting the efficiency of the color propagation step. 

### Multistep Algorith

```c
MULTISTEP(G(V,E))
T = SimpleTrim(G)
V = V exclude T
Select v in V for which din(v)*dout(v) is maximal
D = BFS(G(V,E(V)), v)
S = D intersection BFS(G(D,E'(D)), v)
V = V exclude S
while NumNodes(V)>n_cutoff do
	C = MS-ColorSCC(G(V,E(V)))
	V = V exclude C
Tarjan(G(V, E(V)))
```

```c
MS-ColorSCC(G(V,E(V)))
for all v in V do in parallel
	Color(v) = v
	add v to Q
  	Visited(v) = false
end for

while Q is not empty do
	for all v in Q do in parallel
		for (u,v) in E(V) do
			if Color(v) > Color(u) then
				Color(u) = Color(v)
				if Visited(u) = false then
					Visited(u) = true
					Add u to Q_t
				end if
			end if
		end for
		if any u changed color then
			if Visited(v) = false then
				Visited(v) = true
				Add v to Q_t
			end if
		end if
	for all v in Q_t do in parallel 
		Visited(v) = false
	end for
	Barrier
	Q = merge all Q_t
end while
```
This is a hybrid algrithm combining serial algorithm, FW-BW algorithm and Coloring Algorithm. It is based on the observation that FW-BW is efficient if a graph has a relatively small number of large and equally-sized SCCs, as the leftover partitions in each step could, on average, result in similar amounts of task-parallel work. Then if a graph has a big SCC and a lot of small SCCs, FW-BW would result in a large work imbalance. Conversely, the coloring algorithm is quite efficient when the graph has a large number of small and disconnected SCCs. The runtime of each coloring step is proportional to the diameter of the largest connected component in the graph. The time for each step can be very high when the largest SCCs remain, and there is no guarantee that these SCCs will be removed in any of the first few iterations. When the number of nodes in the graph is less than a threshold, all parallel algorithms perform poorly over serial algorithm like Tarjan's algorithm because of parallel overhead.

Multistep uses FW-BW to find the largest SCC first and then use Coloring algorithm to find the remaining small SCCs. Last, after the number of nodes drop to certain threshold, it uses serial algorithm. 

Note, MS-Coloring can be run asynchronously. 

### UFSCC algorithm

```c
UFSCC code for worker p
for every v in V
	S(v) = {v}
end for
DEAD = DONE = empty set
for every worker p in P
	R_p = empty stack
end for
for each worker p in P
	UFSCC_p(v0)
end for
```

```c
UFSCC_P(v)
R_p.push(v)
while v' in S(v) \ DONE 
	for each w in RANDOM(post(v')) do
		if w in DEAD then continue
		else if there is not w' in R_p: w in S(w') then
			UFSCC_p(w)
		else while S(v)!=S(w) do
			r = R_p.pop
			UNITE(S,r, R_p.TOP())
		end if
	end for
	DONE = DONE union {v'}
	if S(v) not in DEAD
		DEAD = DEAD union S(v)
		report SSC S(v)
	end if
	if v = R_p.TOP()
		R_p.pop()
end while
```

The strongly connected components are tracked in a collection of disjoint sets (union-find data structure), which we represent using a map: S: V => 2^V with the invariant: for any v, w in V: w belongs to S(v) <=> S(v) = S(w). We denote an edge (v, v') in E as v -> v'. post(v) is a successors function: post(v) = {w|v -> w}. S(a) \ S(b) will get a set of elements in S(a) but not in S(b). \ means excluding here. UNITE function on S mergers two mapped sets. 

The algorithm of each work is based on DFS and Set operations. It use recursion to push connecting nodes and when it pushes a node already in stack, it forms a cycle then is a SCC (because SCC are formed with cycles). More strict proof of its correctness is in its [paper](https://dl.acm.org/citation.cfm?id=2851161). Anyway, UFSCC_P itself can find a (partial) SCC. Then P workers randomly process th graph starting from v0 and prune each other's search space by communicating parts of the graph that have been processed.


## Cost of Adaptive CC and Soman CC

Define a graph G(V,E) where V is all the nodes in the graph G and E is all the edges in the graph G. Suppose there are P processors. 

### Communication volumn of Soman CC

In Soman CC, let's suppose edges and nodes are distributed on P processors, and at least one end of the edge is on the processor who owns that edge. S iterations are required to finding all connected components. In each iteration, each edge tries to hook, then each node does multi-pointer-jumping. Before next iteration, the nodes on the border of each partition exchange information (connected component ID). In the worst case, each partition need to communicate with all nodes who don't reside in that partition, thus the communication volumn is (P-1)\*|V|\*S in total. Usually only the nodes on the border of each partition need communication, but how to estimate the size of border at random partition case? A good feature about Soman CC is that we know S is that it is log|V| generally, usually a small constant (log1000000000=18). In empirical experiments, S is between 2 to 5. 

Assumption: 

	1) worst case, each partition need to communicate with all nodes on other partition.

	2) S is small constant, say 5

		communication volumn: O((P-1)|V|)

		time for communication: Max( 5\*latency, 5(P-1)|V|/bandwith\*latency )

### Communication volumn of Adaptive CC

In Adaptive CC, all edges |E| are tring to hook and each hook operation may result in a traversal across partitions because either end of the edge will traverse back to root and hook with the lower-than-root-ID node along the parent path of the other end of that edge, which also makes the resulting tree couldn't be very high. So in the worst case, every hook operation results a traversal in multiple partition. Here we assume, each hooking will hook the lowers tree to the higher tree. Then from MST, the depth of the tree is bounded by log\*|V| which is less than 5. So number of partitons traversed is bounded by log\*|V|. Total communication volumn for hooking is |E|log\*|V| < 5|E|. 

Same story applies to multiple pointer jumping, each node try to do pointer jumping and in the worst case, every pointer jumping results in a traveral in log\*|V| partition. So the total communication volumn for pointer jumping is |V|log\*|V| < 5|V|. Total communication volume is bounded by O(|E|+|V|).

An interesting thing is, in actual implementation, the edges are divided into 2|E|/|V| segments. Then in each segment, there are |V|/2 edges trying to hook, either end of the edge will traverse back to root and hook with the lower-than-root-ID node along the parent path of the other end of that edge. So in the most idea case, those |V|/2 hooks don't traverse cross any partitions. So for each segments, there is no communication cost. In the worst case, each edge traverses log\*|V| partitions, however they can be overlapped by each other somehow. We define OM is amount of messages can be sent at the same time. Also we assume, each communication happens in a batch style, OM amount of messages are sent simultaneously and each OM messages are sent follow by follow. Then if they are perfectly overlapped, they happens at the same time and can be sent once, then OM is log\*|V|\*|V|/2, the time used for communication is only (log\*|V|\*|V|/2)/bandwith\*lantency, even though the communication volum is still log\*|V|\*|V|/2. 2\*|E|/|V| of those communications has to be in sequence. It is interesting to estimiate how much of the communication can be overlapped. Comparing to the Soman CC, there is (P-1)\*|V| volumn of communication between each iteration, but they can be transferred once, so the time for communication is (P-1)\*|V|/bandwith\*latency. 

It makes sense to argue hooking the shorter tree to taller tree is a strong assumption, then let's relax it to a traversal will result only O(P) partition traversal. Then for each segments, for hooking step, it results in O(P|V|) communication volumn, for multi-pointer-jumping, it results in O(P|V|) communication volumn. So for all segments, it results in O(P|E|) communication volumn. Because segments are processed in sequence, the time for communication is Max(P|V|/OM\*latency\*2|E|/|V|, P|V|/bandwidth\*latency\*2|E|/|V|).

Assumption:

	1) Hooking process always hooks the shoter tree to taller tree. Then depth of the tree resulting from hooking is bounded by log\*|V|

	2) We define OM is the amount of messages can be sent simultaneously and each OM amount of messages are sent one by one untill all the messages are sent. 

		communication volumn: O(|V|+|E|)

		time for communication: Max(7.5*|V|/OM*latency*2*|E|/|V|, 7.5*|V|/bandwidth*latency*2*|E|/|V| ). When OM < bandwidth, we take the formal one. When OM > bandwidth, we take later one.

	3) If we assume the traversal resulted from hooking and pointer jumping will cross O(P) partitions:

	communication volumn: O(P(|V|+|E|))

	time for communication: Max(1.5|V|P/OM*latency*2|E|/|V|, 1.5|V|P/bandwidth*latency*2|E|/|V|). When OM < bandwidth, we take the formal one. When OM > bandwidth, we take the later one.

From above, we know if the computation is large enough to generate enough communication to saturate the bandwith, asyn's communication is the same as syn's communication with a larger constant. Then they need to compete on which is smaller: 2|E|/|V| or S.  

### Cost of Async and Sync

Async = C + M + AT, where C is computation cost, M is communication cost and AT is atomic cost

Sync = C' + M' + SC, where C' is computation cost, M' is communication cost, and SC is synchronization cost.

C' < C, C is O(E) and C' is O(|E|\*S), S is constant from experiment experience. 

M and M' is hard to compare, 1) for Soman, M' = S(P-1)|V|/bandwith\*latency. Say P is 11, then if bandwith is larger, M' is not significant. Anyway, M'is O(|V|). 2) for adaptive CC, M = (1.5\*|V|P/OM\*latency)\*2\|E|/|V|. If OM (overlap amount) is |V|, then M is O(|E|/|V|).  Since the graph is scale-free, then |E| < |V|^2 and the communication under the condition that overlap ratio is |V|. However we don't know how much of the work can overlap.  

AT = |E|/amount of overlapped atomic operation * t, where t is the time to finish an atomic operation. In adaptive CC, in each segment, there are |V|/2 hookings working on |V| memory space. Then if the ratio of overlapped atomic operation is |V|, AT is 2|E|/|V|\*t. 
SC = S * s, where s is the time cost for synchronization. We assume S is less 5. 

AT and SC are also hard to compare, since we can only say s and t are some constant. How many unoverlapped atomic operation, actually, the unoverlapped atomic operation can be sum of degree of all nodes in the worst case. But I don't think it happens often. Actually I don't think it can be so unlucky. 

There is another level of overlapping in Async case. Unlike sync CC, the boundary of computation, communication and synchronization are clear. computation, communicaiton and atomic operations are overlapped to each other, then may shrink the overall runtime. However, the amount of the overlap is unknown.

Summary: 

	Syn = S*C + S*((P-1)|V|/bandwith*lantency) + S*s


	Asyn = C + Max( 1.5|V|P/OM*lantency*2|E|/|V|, 1.5|V|P/bandwidth*latency*2|E|/|V|) + |E|/amount of overlap atomic operations*t

I think let's run simulations.

### DFS on GPU

I read that DFS on GPU paper, it uses three BFS and some scheme to do DFS. Then we can not use Cong's DFS asynchronous CC on GPU using that DFS-ON-GPU techique since BFS and asynchronous seem contradict to each other. In DFS-ON-GPU paper, the BFS uses iteration synchronization. Not what we want. Pass.

## Irregular Algorithms?? Ordered or unordered??

Many problems are irregular since they use pointer-based data structures such as trees and graphs. So how's the structure of parallelism and locality in irregular algorithm? A major complication is that dependences in irregular algorithm

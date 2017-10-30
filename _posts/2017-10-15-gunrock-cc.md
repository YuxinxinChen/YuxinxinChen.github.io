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

Unfortunately, the memory access pattern of SV is unfovorable for implementation on a distributed memory system. The "shortcutting" step accesses the grandparent of a vertex u stored as D[D[u]], where D represents the parent relationship. When D is distributed among processors, accessing D[D[u]] generates erratic remote access. In addition, there will be a flood of messages going to the processor that owns the root of the tree as the root is accessed by each "pointer jumping". In addition, for both BFS and SV, the use of global barriers caues scaling problem when many processors are available. SV takes O(log n) barriers, while parallel BFS needs O(d) barriers. 


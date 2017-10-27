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
Then the Parent label will propagate from the vertex with min id to the vertex whose distance is 1 from min-id vertex, then distance 2 ....., farest vertex away from min-id vertex, like a wave. In the worst case, n nodes form a path with min-id in either of end, the runtime will be O(n). But generally, the resulting propagetion will form a tree whose depth is O(log(n)), then the runtime usually is O(log(n))

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
	for each tree do in parallel:
		multi pointer jumping
end while
```
In above version, the algorithmic complexity is the same as SV version, the sequence we do hooking and pointer jumping doesn't change algorithmic complexity. In stead of doing a single pointer jumping after a parallel hooking, we do a iterative pointer jumping till that tree becomes a star tree. Then the hooking after multi pointer jumping is still the same as the level-1 hook in SV version. However, in the worst case, the while will need O(log(n)) iterations and each multi pointer jumping will have O(log(n)) iterations, then the runtime is O(log(n)^2). Seems Soman has a worse runtime, but it is favored by GPU since multi pointer jumping can be written into one kernel and has better memory access coalease and less branch divergence. Think about in this way, we aggregate all the possible pointer jumping together, then each memory fetch (several mem cache lines) will get better utilized in terms of how much pointer jumping can do comparing to SV version. At the same time, because it need to jump more than once, the ratio of runtime of the two divergent threads is large. Hence thread serialization penalty was minimized. However, in Soman's paper, he didn't compare the Soman version with SV version but only compare Soman's GPU version with Soman's CPU version. Shame. Anyway, seems everyone is using Soman version CC for GPU, it might just win because of its hardware utilization is higher then it runs faster.




Unfortunately, the memory access pattern of SV is unfovorable for implementation on a distributed memory system. The "shortcutting" step accesses the grandparent of a vertex u stored as D[D[u]], where D represents the parent relationship. When D is distributed among processors, accessing D[D[u]] generates erratic remote access. In addition, there will be a flood of messages going to the processor that owns the root of the tree as the root is accessed by each "pointer jumping". In addition, for both BFS and SV, the use of global barriers caues scaling problem when many processors are available. SV takes O(log n) barriers, while parallel BFS needs O(d) barriers. 


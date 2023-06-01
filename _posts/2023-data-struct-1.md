### std::map VS. std::unordered_map

Different libraries may employ diverse strategies in their implementations. In this discussion, we will narrow our focus to the source of stdlibc++.

The std::map, as part of its underlying implementation, utilizes a Red-black tree. This is a specific type of binary tree characterized by an additional bit signifying 'red' or 'black.' The purpose of this extra bit is to rearrange the tree to achieve better balance. Employing a binary tree is a logical choice considering a std::map can be traversed in order - an operation that would be less efficient if performed with a hash map.

On the other hand, std::unordered_map employs a hash table (which includes a linked list for each bucket) in its underlying implementation. The rationale for using a hash table is that a std::unordered_map does not offer the capability to traverse in order. Therefore, the standard library opts for a hash map instead of a Red-black tree, as the former provides superior amortized time complexity for insert operations.

The decision to use either map or unordered_map is largely determined by whether the data being inserted needs to be sorted.

```
                      map            unordered_map
Insert               log(n)              1
Find any             log(n)              1    
Find max/min           1                 n (checking if a bucket is empty for all slots in the hash table also may give you a big constant)
Delete               log(n)              1
```
### std::set VS. std::unordered_set

The std::set, as part of its underlying implementation, utilizes a Red-black tree. Employing a binary tree such as Red-black tree is a logical choice considering a std::set can be traversed in order - an operation that would be less efficient if performed with a hash map.

On the other hand, std::unordered_set employs a hash table (which includes a linked list for each bucket) in its underlying implementation. The rationale for using a hash table is that a std::unordered_set does not offer the capability to traverse in order. Therefore, the standard library opts for a hash map instead of a Red-black tree, as the former provides superior amortized time complexity for insert operations.

The decision to use either set or unordered_set is largely determined by whether the data being inserted needs to be sorted.

```
                      set            unordered_set
Insert               log(n)              1
Find any             log(n)              1    
Find max/min           1                 n (checking if a bucket is empty for all slots in the hash table also may give you a big constant)
Delete               log(n)              1
```
### set (unordered_set) VS. map (unordered_map)

Set(unordered_set) and map (unordered_map) are extremely similar. They are designed and separated for different using niche. The map stores pairs of (key, value), wile set stores only keys.


### Heap VS. Binary Search Tree (BST)

Heap is a kind of binary tree. However, Heap just guarantees that elements on higher levels are greater (for max-heap) or smaller (for min-heap) than elements on lower levels, whereas BST guarantees order (from "left" to "right"). If you want sorted elements, go with BST

Heap is better at findMin/findMax (O(1)), while BST is good at all finds (O(logN)). Insert is O(logN) for both structures. If you only care about findMin/findMax (e.g. priority-related), go with heap. If you want everything sorted, go with BST.

```
          Type      BST (*)   Heap
Insert    average   log(n)    1
Insert    worst     log(n)    log(n) or n (***)
Find any  worst     log(n)    n
Find max  worst     1 (**)    1
Create    worst     n log(n)  n
Delete    worst     log(n)    log(n)
```
All average times on this table are the same as their worst times except for Insert.

- *: everywhere in this answer, BST == Balanced BST, since unbalanced sucks asymptotically
- **: using a trivial modification explained in this answer
- ***: log(n) for pointer tree heap, n for dynamic array heap

#### Advantages of binary heap over a BST

- average time insertion into a binary heap is `O(1)`, for BST is `O(log(n))`. This is the killer feature of heaps.

There are also other heaps which reach O(1) amortized (stronger) like the [Fibonacci Heap](https://en.wikipedia.org/wiki/Fibonacci_heap), and even worst case, like the [Brodal queue](https://en.wikipedia.org/wiki/Brodal_queue), although they may not be practical because of non-asymptotic performance: [link](https://stackoverflow.com/questions/30782636/are-fibonacci-heaps-or-brodal-queues-used-in-practice-anywhere)

- binary heaps can be efficiently implemented on top of either dynamic arrays or pointer-based trees, BST only pointer-based trees. So for the heap we can choose the more space efficient array implementation, if we can afford occasional resize latencies.

#### Advantages of BST over a binary heap
- search for arbitrary elements is `O(log(n))`. This is the killer feature of BSTs.

For heap, it is `O(n)` in general, except for the largest element which is `O(1)`.

#### Average binary heap insert is `O(1)`
Sources:

- Paper: [link](http://i.stanford.edu/pub/cstr/reports/cs/tr/74/460/CS-TR-74-460.pdf)
- [WSU slides](https://web.archive.org/web/20161109132222/http://www.eecs.wsu.edu/~holder/courses/CptS223/spr09/slides/heaps.pdf)

Intuitive argument:

- bottom tree levels have exponentially more elements than top levels, so new elements are almost certain to go at the bottom
- heap insertion starts from the bottom, BST must start from the top

In a binary heap, increasing the value at a given index is also `O(1)` for the same reason. But if you want to do that, it is likely that you will want to keep an extra index up-to-date on heap operations: [link](https://stackoverflow.com/questions/17009056/how-to-implement-ologn-decrease-key-operation-for-min-heap-based-priority-queu) e.g. for Dijkstra. Possible at no extra time cost.

### Insertion Performance Showcase

![](../images/data_struct1.png)

[benchmark code](https://github.com/cirosantilli/linux-kernel-module-cheat/blob/52a203a1e22de00d463be273d47715059344a94b/userland/cpp/bst_vs_heap_vs_hashmap.cpp)
[plot script](https://github.com/cirosantilli/linux-kernel-module-cheat/blob/52a203a1e22de00d463be273d47715059344a94b/bst-vs-heap-vs-hashmap.gnuplot)

So clearly:

- heap insert time is basically constant.

- We can clearly see dynamic array resize points. Since we are averaging every 10k inserts to be able to see anything at all above system noise, those peaks are in fact about 10k times larger than shown!

- The zoomed graph excludes essentially only the array resize points, and shows that almost all inserts fall under 25 nanoseconds.

- BST is logarithmic. All inserts are much slower than the average heap insert.

#### BST cannot be efficiently implemented on an array
Heap operations only need to bubble up or down a single tree branch, so `O(log(n))` worst case swaps, `O(1)` average.

Keeping a BST balanced requires tree rotations, which can change the top element for another one, and would require moving the entire array around (`O(n)`).

#### Heaps can be efficiently implemented on an array
Parent and children indexes can be computed from the current index as [shown here](http://web.archive.org/web/20180819074303/https://www.geeksforgeeks.org/array-representation-of-binary-heap/).

There are no balancing operations like BST.

Delete min is the most worrying operation as it has to be top down. But it can always be done by "percolating down" a single branch of the heap as [explained here](https://en.wikipedia.org/w/index.php?title=Binary_heap&oldid=849465817#Extract). This leads to an `O(log(n))` worst case, since the heap is always well balanced.

If you are inserting a single node for every one you remove, then you lose the advantage of the asymptotic O(1) average insert that heaps provide as the delete would dominate, and you might as well use a BST. Dijkstra however updates nodes several times for each removal, so we are fine.

#### Dynamic array heaps vs pointer tree heaps

Heaps can be efficiently implemented on top of pointer heaps: [link](https://stackoverflow.com/questions/19720438/is-it-possible-to-make-efficient-pointer-based-binary-heap-implementations)

The dynamic array implementation is more space efficient. Suppose that each heap element contains just a pointer to a struct:

- the tree implementation must store three pointers for each element: parent, left child and right child. So the memory usage is always 4n (3 tree pointers + 1 struct pointer).

Tree BSTs would also need further balancing information, e.g. black-red-ness.

- the dynamic array implementation can be of size 2n just after a doubling. So on average it is going to be 1.5n.

On the other hand, the tree heap has better worst case insert, because copying the backing dynamic array to double its size takes O(n) worst case, while the tree heap just does new small allocations for each node.

Still, the backing array doubling is O(1) amortized, so it comes down to a maximum latency consideration. [Mentioned here](https://stackoverflow.com/questions/19720438/is-it-possible-to-make-efficient-pointer-based-binary-heap-implementations/41338070#41338070).

### Philosophy

- BSTs maintain a global property between a parent and all descendants (left smaller, right bigger).
The top node of a BST is the middle element, which requires global knowledge to maintain (knowing how many smaller and larger elements are there).
This global property is more expensive to maintain (log n insert), but gives more powerful searches (log n search).

- Heaps maintain a local property between parent and direct children (parent > children).
The top note of a heap is the big element, which only requires local knowledge to maintain (knowing your parent).

Comparing BST vs Heap vs Hashmap:

- BST: can either be either a reasonable:
	- unordered set (a structure that determines if an element was previously inserted or not). But hashmap tends to be better due to O(1) amortized insert.
	- sorting machine. But heap is generally better at that, which is why heapsort is much more widely known than tree sort

- heap: is just a sorting machine. Cannot be an efficient unordered set, because you can only check for the smallest/largest element fast.

- hash map: can only be an unordered set, not an efficient sorting machine, because the hashing mixes up any ordering.

#### Doubly-linked list
A doubly linked list can be seen as subset of the heap where first item has greatest priority, so let's compare them here as well:

- insertion:
	- position:
		- doubly linked list: the inserted item must be either the first or last, as we only have pointers to those elements.
		- binary heap: the inserted item can end up in any position. Less restrictive than linked list.

	- time:
		- doubly linked list: O(1) worst case since we have pointers to the items, and the update is really simple
		- binary heap: O(1) average, thus worse than linked list. Tradeoff for having more general insertion position.

- search: O(n) for both

An use case for this is when the key of the heap is the current timestamp: in that case, new entries will always go to the beginning of the list. So we can even forget the exact timestamp altogether, and just keep the position in the list as the priority.

This can be used to implement an [LRU cache](https://stackoverflow.com/questions/23772102/lru-cache-in-java-with-generics-and-o1-operations/34206517#34206517). Just like for [heap applications like Dijkstra](https://stackoverflow.com/questions/14252582/how-can-i-use-binary-heap-in-the-dijkstra-algorithm), you will want to keep an additional hashmap from the key to the corresponding node of the list, to find which node to update quickly.


Notes Resource links: 
https://cs.stackexchange.com/questions/27860/whats-the-difference-between-a-binary-search-tree-and-a-binary-heap


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


### Heap VS. Binary Tree

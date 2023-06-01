### std::map VS. std::unordered_map
Different libraries may have different implementations. Let's only focus on stdlibc++ source here (also apply to the following discussion)

std::map uses Red-black tree (a special case of binary tree with one more bit representing red or black. The extra bit is used to rearrange the tree into more balanced tree.) as the underlying implementation. The use of binary tree makes sense, since std::set can be traversed in order, which would not be efficient if a hash map were used.

std::unordered_map uses hash table (hash table + linked lisk for each bucket) as the underlying implementation. The use of hash table makes sense, since std::unordered_map cannot be traversed in order, so the standard library chose hash map instead of Red-black tree, since hash map has a better amortized insert time complexity.

If the inserted data is required to be sorted or not leads to decision using map or unordered_map.

```
                      map            unordered_map
Insert               log(n)              1
Find any             log(n)              1    
Find max/min           1                 n (checking if a bucket is empty for all slots in the hash table also may give you a big constant)
Delete               log(n)              1
```
### std::set VS. std::unordered_set

std::set uses Red-black tree (a special case of binary tree with one more bit representing red or black. The extra bit is used to rearrange the tree into more balanced tree.) as the underlying implementation.
std::unordered_set uses hash table (hash table + linked lisk for each bucket) as the underlying implementation.




### set (unordered_set) VS. map (unordered_map)


### Heap VS. Binary Tree

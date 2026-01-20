+++
date = '2026-01-20T13:15:58+11:00'
draft = true
title = 'LFU Cache'
+++

While reading on LRU, I found about LFU caches and was intrigued to see how I can try to make my own.

### Design choices
- Interface-wise, the Least Frequently Used (LFU) cache is similar to Least Recently Used (LRU). 
  - `get(key)`
    - Increase frequency of key-value entry, return value.
  - `put(key, value)`
    - Increase frequency, update value.
    - If at capacity, evict an entry.
- The difference with LRU is the eviction policy. This time, the entry that is the least frequently accessed/updated is evicted in constant time. If there are multiple entries with the same frequency, use LRU rules to evict.

- Based on the previous LRU project, we know that we can use `list` to accomodate the LRU logic. Since we already had the LRU base, we should try to isolate the LRU logic and find another way to enforce the LFU logic by going to a higher level. Turns out, we could do this by grouping nodes into lists of different frequencies. Let's see how we can make this method work.

### Multiple Lists

**Get**
- When we hear O(1) access based on a key, our first instinct is to grab an `unordered_map` which does give an average O(1) access. Paired with an iterator to the `list` node above which allows us to directly access the node, we can ensure a fast access to our node. 
- Does increasing the node frequency is also an O(1) operation? Yes! 
  - Removal from a list is O(1), and since we have the frequency variable in the node, we can easily create/access the list at frequency+1, and add a node at O(1) as well

**Put**
- O(1) for `put` is achieved through the same `unordered_map`. 
  - If the node exists, we create a new node in a frequency+1 `list` and delete the old one.
  - If the node does not exist:
    - And if the cache is at capacity:
      - we remove a node at the lowest frequency. We use a `min_freq` variable to track this. If there is more than one node in the list, we pop the least recently used.
    - Then we add the new node to the `list` where frequency == 1.

### Why not other data structures?
- `map` operations are `O(logN)`, so they are not considered.
- `vector`, `deque` and `list` provide O(1) removal from their front/end, hence they can be considered.

### Node
- At first, `value` is the only member attribute in the node. We then add the frequency as a member so that we can know which list to move the node to when its frequency increases. Alternatively, we could put this information in the map eg. `unordered_map<T, std::pair<int, list_iterator>>`, however we decided to keep it in the node because it is easier to read and reason with.
- We also added key inside the node, which is to enable us to know which entry in the map to delete whenever we `pop` the least frequently used node. Without this key, we will have no way to update the map.

### Implementation

Source code can be found [here](https://github.com/mariusandrian/CPP-LFU-Cache)

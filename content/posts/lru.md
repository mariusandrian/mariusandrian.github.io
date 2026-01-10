+++
date = '2026-01-10T14:04:34+11:00'
draft = true
title = 'LRU Cache'
+++

While studying data structures, I came across the LRU Cache and wanted to make my own implementation. It is a key-value store that performs in a way so that the most recently accessed items remain in the cache. Here, I noted some of my design decisions and considerations.

## Design
We first create the underlying data structures for the LRU cache. 
The requirements for the cache:
- O(1) retrieval of values anywhere in the cache using a key.
- O(1) setting of values which include cache eviction.
- Memory should have a fixed size.

Since we are going to use key to associate to a value, we opted to use std::unordered_map to ensure an average of O(1) lookup.
The key could be of any hashable type, and the value is the iterator to the std::list.

We use a std::list to hold our values because it allows O(1) for insertion/removal anywhere in the container.
This consequently allows us to promote a value in O(1). The list's size can be managed by our user defined class.   

Other alternatives considered:

* `std::forward_list`
  
    Why do we need a bidirectional pointer? Maybe we can simply use a singly linked list.
    In forward_list, it is true that we will save some memory from only holding the next pointer. However, the class doesn't have any `pop_back` in its API. While it is technically possible to manage the last and the 2nd last iterator to enable a `pop_back` behavior, this is extra work that can be avoided.

* `std::vector`

    This would not work because promoting a value to the top of the list would require O(N) to push everything else down.

* `std::deque`

    Deque does have `push_front` and `pop_back` which are guaranteed O(1). 
    However, deque suffers from O(N) when promoting a value to the top of the list. This is due to its nature of being a sequence of contiguous arrays. 
    Removing a value necessitates scooting elements after it forwards - similar problems to a std::vector.

    Another problem with deque is its iterator invalidation when `push_front` and `pop_back` is done.
    This would prevent us from using our map to access values directly and quickly.

### ListNode struct

Originally, the list only contains the data of type T. However, this posed a problem when evicting the last element in a list.
When we evict, we also need to remove said element from the map lest we have an ever-growing map.
However, the list only holds the data T, but not the key. We need to include the key inside the list node itself to properly remove the key from the map as well.
This does create a duplicate key, however at this point the memory overhead is less important than the O(1) requirement.

### Result

This is the data structure chosen:


```c++
template <typename K, typename T>
class LRUCache
{
private:
    std::list<ListNode<K, T>> list_;
    std::unordered_map<K, iterator> map_;
    size_t capacity_;
};
```

Why do we use templates? Since I'm learning Standard Template Library (STL) in the meantime, it makes sense to learn it as well..

It's also cool to see it being able to take in different types.

## Algorithm
1. **Inserting values**
   
    We first check whether the cache is full or not. If it is, we evict the last recently used element and remove that element from our map.

    We then put the new element at the front of our list. We update the map with the new key and its iterator which is at the front of the list.


2. **Getting values**
   
    We check the existence of the key in the map, and return an empty optional if it's not found.

    If it is found, we promote the element to the top of the list and return the data back to the caller.

## Source

The source code can be found in this [github repo.](https://github.com/mariusandrian/cpp-lru-cache)

## Future

Next up is to handle multithreading. I hope to add some stress tests and benchmarks to prove its efficiency.
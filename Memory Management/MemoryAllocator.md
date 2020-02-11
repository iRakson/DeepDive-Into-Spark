# Memory Allocator

Actual Memory Allocation is done by Memory Allocators. We have two different allocators for allocating memory in off-Heap
and on-Heap mode.

* **Unsafe Memory Allocator** :
It is responsible for allocating off-Heap memory. It internally calls unsafe's *allocateMemory()* for allocating memory 
and *freeMemory()* for freeing the memory. If memory debug is enabled, allocated memory bytes are filled with `0xa5` and 
de-allocated bytes are filled with `0x5a`.

* **Heap Memory Allocator** : 
It is responsible for allocating on-heap memory. It uses long array for allocating the memory while JVM's Garbage collection
takes care of freeing the unused memory. Spark uses pooling mechanism for allocating memories from on-heap memory. It 
maintains a pool which keeps track of all the allocated but not used memory blocks and whenever a new request comes, it
first check whether any block of that much memory size is available or not. If its available then that block is returned
otherwise a new memory block is asked from long array.

Both these allocators are implemented in *org.apache.spark.unsafe.memory*. Both of them implements *MemoryAllocator*.
Next chapter -> [Memory Consumer](MemoryConsumer.md)

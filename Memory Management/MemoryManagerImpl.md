### How does spark implements this memory management:

It has an abstract class named MemoryManager which has methods for acquiring memory, releasing memory, checking memory 
uses etc. This class is extended by the two types of memory manager that we discussed above StaticMemoryManager and 
UnifiedMemoryManager. These two classes will implemetnt their logic for acquiring and releasing execution, storage and 
unroll memory. For bookkeeping about all the available memory, spark has another abstract class called 'MemoryPool'. 
There are 2 classes(ExecutionMemoryPool and StorageMemoryPool) that extends this abstract class. As spark has to do 
bookkeeping for on-heap memory as well as off-heap memory, it creates total 4 pools.

These classes do not actually allocate memory to the requesting memory consumers(we will in detail about them later). 
They only decide the boundary for execution and storage memories and decide how much memory to grant for each 
acquirememory request.

Insert a pic depicting the relation between the memorypool, memorymanager and memory consumer. Also insert a pic showing
how memory is actually divided in both strategies. 

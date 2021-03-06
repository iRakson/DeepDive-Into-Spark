# Spark Memory Manager :

Spark Memory Manager is responsible for managing the execution and storage memory. We have one memory manager for each 
JVM. There are two approaches of managing the memory across execution and storage. Either we can fix size for both of 
them or be a little flexible and dynamically change their size depending on the spark application.

Spark used the first approach to start with and wrote `StaticMemoryManager` which fixes the size for execution and storage.

* **Static Memory Manager** : Spark used this memory manager by default until spark 1.6. Static Memory Manager allocates
fixed amount of memory to execution and storage. Although spark provides configurations to change these sizes before application
starts running. The major drawback here was that even if an application don't need any storage memory, it can't use the 
fraction of reserved storage memory and vice versa. To use this static memory manager after spark 1.6 we need to switch 
legacy configuration.

* **Unified Memory Manager** : The major drawback of static memory manager was that it allocates fixed size to execution
and storage. Unified memory manager tries to solve this problem by dynamically allocating memory to execution and storage.
It maintains a soft boundary which keeps changing depending on the spark application memory requests. Spark proposed this
solution in spark 1.6 and since then Unified Memory Manager is used as default memory manager. 

![Spark Memory Management 1.6+](Spark-Memory-Management-1.6.0.png) 

We have three regions of memory available to us as we discussed earlier. Apart from reserved memory we have
spark memory and user memory. We can configure the spark memory using `spark.memory.fraction`. By default it is set to
0.6. Initially default was 0.75 but later it changed to 0.6 to improve the performance of spark application.

* Off-Heap memory is only divided into 2 regions. One for execution and other for storage. Their size is decided by
`spark.memory.storageFraction`.

By default, we set a soft boundary at 50%(using configuration spark.memory.storagefraction) i.e. both storage and execution
can take 50% each of total spark memory.  
Lets understand this soft boundary by two scenarios:

* Our application is storage intensive and it caches a lot of data. And lets assume already it has used 70% of total 
spark memory. Now what will happen if a request for execution memory comes ? Spark will first check whether it can expand
the boundary of execution memory which it certainly can as soft boundary is set to 50% and our storage is using more than
its fair share of memory. So some of the cached data will be evicted to disk and required amount to space will be provided
for execution.

* Our application is execution intensive and it uses a lot execution memory and lets assume that it has already used 
70% of total memory. So now what will happen if a new request for storage memory comes? Well first spark will check 
whether it has any amount of storage memory left, it its available then it will use that memory. What if there is no 
storage memory left as well ? Now what spark will do ? As we remember the soft boundary is set to 50 % and execution is 
using more than its fair share of memory. So some of execution data should be evicted to disk as we have seen in above case.
Spark handle it other way though, it evicts some of the storage data to disk and then deals with the current request for
storage memory. The main argument behind this approach is that if we evict some of the execution data to disk then its atmost 
certain that it will be brought back into memory to complete the execution of application while same cant be said for the
storage data. It may or may not be brought back into the memory. Also while evicting the execution data is pretty 
complex process.

So this means that execution will always acquire any amount of memory ? What with application which requires to have a 
lots of cache and also needs lots of execution memory as well, such as ML applications. Well here 'spark.memory.storagefraction' 
comes into play. Execution area can't be expanded beyond this boundary if our application has already acquired that much
storage area. 

* Until spark 2.4 both static memory manager and unified memory manager were there for users. From spark 3.0, only unified
memory manager is available for users.

## How does spark implements memory management:

Spark defines a abstract class named _MemoryManager_ which has methods for acquiring memory, releasing memory, checking 
memory uses etc. This class is extended by two types of memory manager that we discussed above _StaticMemoryManager_ and 
_UnifiedMemoryManager_. These two classes will implement their logic for acquiring and releasing execution, storage and 
unroll memory. For bookkeeping about all the available memory, spark has another abstract class called _MemoryPool_. 
There are 2 classes(_ExecutionMemoryPool_ and _StorageMemoryPool_) that extends this abstract class. As spark has to do 
bookkeeping for on-heap memory as well as off-heap memory, it creates total 4 pools.

Note : Memory manager do not allocate any memory, it only adjusts the boundaries of memory pools and returns the amount 
of memory that can be granted for a request in case of execution memory request and whether to grant requested memory or
not in case of storage memory request. 

## Memory Pool
Memory pools are used for doing the book keeping by the memory managers. Spark has execution memory pool for execution
memory and storage memory pool for storage memory. We have separate pools for off-heap memory and on-heap memory. 

Execution memory pool ensures that each tasks gets its fair share of memory. Each tasks with execution memory request can
get at max 1/N of total memory, where N is number of active tasks.

    Max memory allocated to task, max = 1/N
    Min memory allocated to task, min = 1/2N
    N = Number of active tasks

Whenever a memory request from a task comes to execution memory pool it first add the current task into its task list, if
it's not already present. And notify the other tasks about this task. Then it tries to grow the execution memory pool i.e.
it will check whether storage memory pool is using more than its share of memory. If yes then execution pool will grow 
while storage pool will shrink. 

It then calculates max and min using above equation and checks how much memory it can grant to this current task. If toGrant
memory is less than min memory then it will wait till it can grant atleast min amount of memory. Otherwise memory request
will be granted.

Logic for acquiring storage memory is relatively simpler than acquiring execution memory. On receiving a request for storage
memory, memory manager will check whether storage pool has free memory left or not. If not, it will check whether it can borrow 
some memory from execution pool. If it can't get required amount from execution pool then it will simply evict some of the
cached blocks to the disk.


Until now we have seen how memory is arbitrated between execution and storage and how to arbitrate memory between tasks.
There is one memory arbitration left to discuss which spark does:

* How to manage memory across different operators of a task?

This will be discussed in [TaskMemoryManager](TaskMemoryManager.md)

Who exactly allocates the physical memory then? And who make these memory requests?
We will answer this in next chapter [MemoryConsumer](MemoryConsumer.md).

Reference : https://0x0fff.com/spark-memory-management/
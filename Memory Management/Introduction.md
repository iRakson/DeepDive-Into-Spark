# Memory Management in Spark

Spark is one of the fastest big data processing engine available to us at the moment. It gets all its efficiency by performing
in-memory processing. So its memory management becomes one of the most fascinating topics. Here we will try to dig deep
into spark memory management and will see how it manages its memory across tasks, operators, storage etc. 

## On-Heap Memory and Off-Heap Memory

We have two types of memory available. 
* **On-Heap Memory** - This is typical JVM memory that is utilised by the java applications. Spark also runs on jvm so 
on-heap memory is default for spark. JVM will be responsible for the GC and other allocation stuffs. The major drawback 
is GC time.

* **Off-Heap Memory** - Spark also supports off-Heap mode of memory. To use this memory, user has to explicitly turn this 
on by using `spark.memory.offheap.enabled` and also has to set `spark.memory.offheap.size` to positive value. Spark 
implemented its memory allocator for managing the off-heap memory as JVM will not be responsible for managing this memory.

### How is memory distributed across various regions:

First of all lets see how many different regions of memory are there for a spark application:

* **Reserved Memory** : This has been hard-coded to 300MB by spark from spark 1.6.

This memory has been reserved for system. This is done specially to prevent the out of memory error when running the 
spark in local mode. In local mode both driver as well as executor runs as same JVM processes.


* **Spark Memory** : This region is used by the spark applications performing all the execution and caching operations. Size
of this region is configurable and be tuned by user according to their needs. User can tune this memory using `spark.memory.fraction`.
Spark memory itself has been divided into 2 parts : a.) Execution memory b.) Storage Memory

* _Execution Memory_ :
This area of memory is used for all the shuffle, join, sorting and aggregation operations or for storing any intermediate
data during execution. Whole spark memory can be used for execution purpose as it is very difficult evict part of execution
memory to disk. And it is also certain that, evicted data will be brought back into memory for the completion of application.

* _Storage Memory_ :
This area is used for storing all the cached data and for data transfer purposes. Some part of storage memory is used as
_unroll memory_.

Spark memory manager is responsible for managing the spark memory i.e. execution memory and storage memory.

* **User Memory** = Total Memory - Reserved Memory - Spark Memory

This memory region is used to store information about all the user defined data structures, classes, methods. It is also used 
for storing the metadata related to the application. Its size is dependent on the size of spark memory. So this is configurable
as well.

Until now we have seen different types and regions of spark memory. We will understand [Memory Manager](MemoryManager.md)
which is responsible for managing execution and storage memory next.


* Project Tungsten
* What types of memory is present in Spark?
* How is memory distribution done?
* What is the use of each part of the memory?
* How execution and storage memory is managed by spark?
* How each task gets their share of memory from the execution memory?
* How each memory consumer within a task gets their share of memory?
* How actually is memory allocated in the spark and how each task keeps track of all its memory?(Talk about physical allocation and paging) - done



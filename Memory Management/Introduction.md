Memory Management in Spark

* Project Tungsten
* What types of memory is presnt in Spark?
* How is memory distribution done?
* What is the use of each part of the memory?
* How execution and storage memory is managed by spark?
* How each task gets their share of memory from the execution memory?
* How each memory consumer within a task gets their share of memory?
* How actually is memory allocated in the spark and how each task keeps track of all its memory?(Talk about physical allocation and paging)

### First of all lets see how many types of memories are supported by spark.

We have two types of memory available. 
a.) On-Heap Memory - This is typical JVM memory that is utilised by the java applications. Spark also runs on jvm so on-heap memory is default for spark. JVM will be responsible for the GC and other allocation stuffs. The major drawback is GC time. 


b.) Off-Heap Memory - Spark introduced off-heap memory later. To use this memory, user has to turn this on by using 'spark.memory.offheap.enabled' and also has to set 'spark.memory.offheap.size' to positive value. Spark implemented its memory allocator for managing the off-heap memory as JVM will not be responsible for managing this memory.

### How is memory distributed :

There are total 3 partition in the on-heap memory :

1.) Reserved Memory = This has been hard-coded to 300MB by spark.

This memory has been reserved for system. This is done specially to prevent the out of memory error when running the saprk in local mode. In local mode both driver as well as executor runs as same JVM process.


2.) Spark Memory = (Total Memory - Reserved Memory) * spark.memory.fraction

Spark memory itself has been divided into 2 parts : a.) Execution memory b.) Storage Memory
This memory will be used for all the spark application related memory usage. By default spark.memory.fraction=0.6. The default value was chosen to prevent the frequent major GC in storage intesive applications.

3.) User Memory = Total Memory - Reserved Memory - Spark Memory

This memory is used to store information about all the user defined data structures, classes, methods. It is also used for storing the metadata related to the application.


### Division of spark memory

Spark memory itself has been divided into 2 parts : 
a.) Execution memory :
This area of memory is used for all the shuffle, join, sorting operations or for storing any intermediate data during execution. Whole spark memory can be used for execution purpose as it is very difficult evict part of execution memory to disk. And it is also certain that , evicted data will be brought back into memory for the completion of application.

b.) Storage Memory :
This area is used for storing all the cached data and for data transfer purposes. Also this memory is used for reserving some space for urolling.


Spark memory manager is responsible for managing the spark memory i.e. execution memory and storage memory.

### Spark Memory Manager :
In Spark 2.4, we have 2 types of memory managers available to us. Static memory manager and unified memory manager. From Spark 3.0, only unified memory manager is available.

a.) Static Memory Manager : Spark used this memory manager by default untill spark 1.6. As the name suggests, memory was divided into fixed chunks and there was no flexibility. The major drawback here was that even if an application dont need any storage memory, it can't use the fraction of memory reserved for storage and vice versa. To use this static memory manager after spark 1.6 we need to switch legacy configuration.

b.) Unified Memory Manager : We have seen the drawback of static memory manager. Instead of reserving fixed amount of memory for execution and storage, it will be better if we allocate them memory as and when need arises and keep a soft boundary which will change depending on the application that is asking for the memory. In other words set boundary dynamically instead of keeping a fixed boundary.

This approach will ensure the optimal uses of memory. 
By default, we set a soft boundary at 50%(using configuration spark.memory.storagefraction) i.e. both storage and execution can take 50% each of tatal spark memory. This can be changed obviously, but execution memory will be given priority over storage when we have to evict some data from memory. 
Lets understand this by two scenarios:

i.) Our application is storage intensive and it caches a lot of data. And lets assume already it has used 70% of total spark memory. Now what will happen if a request for execution memory comes ? Spark will first check whether it can expand the boundary of execution memory which it certainly can as soft boundary is set to 50% and our storage is using more than its fair share of memory. So some of the cached data will be evicted to disk and required amount to space will be provided for execution.

ii.) Our application is execution intensive and it uses a lot execution memory and lets assume that it has already used 70% of total memory. So now what will happen if a new request for execution memory comes? Well first spark will check whether it has any amount of storage memory left, it its available then it will use that memory. What if there is no storage memory left as well ? Now what spark will do ? As we remember the soft boundary is set to 50 % and execution is using more than its fair share of memory. So some of execution data should be evicted to disk as we have seen in aboe case.
Spark handle it other way though, it evicts some of the storage data to disk and then deals with the current request for storage memory. The main argument behind this approach is that if we evict some of the execution data to disk then its atmost certain that it will be brought back into memory to complete the execution of application while same cant be said for the storage data. It may or may not be brought back into the memory. Also while evicting the execution data is pretty complex process.

So this means that execution will always acquire any amount of memory ? What with apllication which requires to have a lots of cache and also needs lots of execution memory as well, such as ML applications. Well here 'spark.memory.storagefraction' comes into play. Execution area can't be expanded beyond this boundary if our application has already acquired that much storage area. 

### How does spark implements this memory management:

It has an abstract class named MemoryManager which has methods for acquiring memory, releasing memory, checking memory uses etc. This class is extended by the two types of memory manager that we discussed above StaticMemoryManager and UnifiedMemoryManager. These two classes will implemetnt their logic for acquiring and releasing execution, storage and unroll memory. For bookkeeping about all the available memory, spark has another abstract class called 'MemoryPool'. There are 2 classes(ExecutionMemoryPool and StorageMemoryPool) that extends this abstract class. As spark has to do bookkeeping for on-heap memory as well as off-heap memory, it creates total 4 pools.

These classes do not actually allocate memory to the requesting memory consumers(we will in detail about them later). They only decide the boundary for execution and storage memories and decide how much memory to grant for each acquirememory request.

Insert a pic depicting the relation between the memorypool, memorymanager and memory consumer. Also insert a pic showing how memory is actually divided in both strategies. 




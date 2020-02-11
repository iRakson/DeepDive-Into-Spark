# Task Memory Manager
Task Memory Manager is responsible for handling all the memory related requests of a Task. All the memory requests are granted
in terms of pages by TaskMemoryManager. It maintains a page table of all allocated memory blocks. This page table maps pages
with their base addresses.

### Paging 
From our previous discussions we know, in off-heap mode only 64 bits are sufficient to address a memory block while in case
of on-heap we require 128 bits. 

To solve the above problem spark encodes the memory addresses into 64 bits for both the modes using following approach:
Use 64 bits addresses as absolute addresses in off-Heap mode. While in on-heap mode use upper 13 bits for page number and 
remaining 51 bits as offset within the page. These page numbers will act as indexes in Page Table which stores base memory 
addresses. Page table is an array of memory blocks. TaskMemoryManager uses a bitset for tracking free pages and also for
getting page numbers.

Since we have fixed 13 bits for page number, that means we can address 8192 pages. So that's the maximum size of page table.
Maximum size of a page is around 17GB. Thus, we can address around 140 GB of memory for a task.


### Page Allocation
Page allocation is done by allocatePage() method of TaskMemoryManager. It first check whether it can acquire requested 
amount of memory, if it can then a memory allocator is called depending on memory mode. Page allocator will return a new
page with the requested memory size. TaskMemoryManager will allocate a page number to this memory block, update the page
table and return the page.


### Acquiring Execution Memory
Execution memory can be acquired by 3 sources:

    * If enough execution memory is free then new memory pages are allocated.
    * If there is not enough execution memory then other consumers will spill to provide pages to requesting consumer.
    * Even after spilling other consumers if requested amount of memory is not free then, memory spill from current consumer
    will happen.


On receiving acquireExecutionMemory request from any memory consumer TaskMemoryManager will first call the _acquireExecutionMemory()_ 
of _memoryManager_. If previous call grants the requested amount of memory then it will return a long indicating the granted 
bytes. If not enough execution memory is available for current task then it will try to get some page from other memory 
consumers. 

Initial approach for getting pages from other memory consumers:
TaskMemoryManger maintains a hashset of all the consumers for a task. So whenever it has to spill, it will traverse the 
hashset and take the first consumer and spill memory from it. 

This approach has two problems:
* The first problem is that if we spill consumer 1 in first time spilling. After a while, consumer 1 now uses 5MB. Then 
consumer 4 may acquire some memory and spilling is needed again. Because we iterate the memory consumers in the same order,
we will spill consumer 1 again. So for consumer 1, we will produce many small spilling files
* The second problem is that we might spill additional consumers. For example, if consumer 1 uses 10MB, consumer 2 uses 
50MB, then consumer 3 acquires 100MB but we can only get 60MB and spilling is needed. We might spill both consumer 1 and
consumer 2. But we actually just need to spill consumer 2 and get the required 100MB.

To solve the above problems spark come up with another optimized approach:
Sort the consumers on the basis of their memory consumption and use TreeMap as data structure for sorted consumers. 
They use _ceilingEntry_ to get the consumer which has least memory use greater than what is requested. If _ceilingEntry_
returns a consumer then that consumer is spilled. If there is no such consumer then they start spilling consumers with
highest memory use.

If still some amount is memory is required then, consumer will call spill on itself and spill some memory. Each consumer
implement their own spill logic.

### Encoding and decoding Page Number and offset
TaskMemoryManager also implements the logic for encoding and decoding page number and offsets. We already have discussed
the encoding logic in [MemoryConsumer](MemoryConsumer.md)  

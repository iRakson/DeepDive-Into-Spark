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

# Memory Location and Memory Block

In case of Off-Heap memory we can directly address memory using 64-bit addresses. Same is not the case in JVM addressed
memory. To address memory we need to know which object it belongs to and it's offset within that object. This means 128 
bits (64 bit for base address and 64 for offset) are required for addressing memory in on-Heap mode. There is need to 
maintain consistency i.e. memory addresses should be of similar size in both the modes. 

From above discussions it's clear that we need a base object reference and an offset to address a memory location. And to 
address a contiguous block of memory we need size of that memory block along with the other two values. Spark has defined 
*MemoryLocation* and *MemoryBlock* which serves respectively as memory location and memory block. MemoryLocation contains
base object reference and offset. MemoryBlock extends MemoryLocation. It also contains *pageNumber* and size.

Note : All the memory requests are granted in terms of pages only in spark. These pages are nothing but MemoryBlock.
Due to this reason only we have page numbers for memory blocks.

# Memory Consumers

Memory consumers are the one's who makes request for memory. They consume memory for performing various shuffle, sort,
hashing operations. Also they only deal with the tungsten memory(It will be discussed in detail later).
Some of the memory consumers are _ShuffleExternalSorter_, _BytesToBytesMap_, etc. All memory consumers extend *MemoryConsumer*.
Each memory consumer writes their own logic for spilling.  *TaskMemoryManager* handles all the memory requests of memory
consumers. They maintain a linked list for tracking all the pages allocated to them and keep a cursor that points to next
byte in current page.

So how Exactly a memory consumer gets MemoryBlock/page ? Whenever a memory consumer wants to store some intermediate record
it first check whether it has any page with sufficient memory available. If its available then it will simply write the
record in that page. Otherwise, it will call *allocatePage()* of TaskMemoryManager. If its possible then TaskMemoryManager
will allocate a page to this memory consumer and stores this allocated memory block reference in page table's pageNumber 
index and finally returns that memory block to consumer. After acquiring a page, memory consumer will update its own list
of pages and set cursor to zero for this new page. Record will get written in this new page and all the other necessary
value will be updated accordingly. To get encoded address of this record memory consumer will call TaskMemoryManager's 
*encodePageNumberAndOffset()*. encodePageNumberAndOffset() takes current page and cursor as parameter and returns the 
encoded 64-bit record address.

### Memory Address Encoding

Memory address encoding is done by *encodePageNumberAndOffset()* of TaskMemoryManager. It takes page and its cursor as
parameter. To perform encoding it performs following steps:
* Left shifts pageNumber by 51 bits.
* Mask the cursor with a 51-bit bitmask
* Use result of above two operations and perform bitwise OR. 
* Resultant value will be returned as encoded address.

In case of Off-Heap Mode though we have to make some considerations. As its clear from above discussion that offset is 
of 51 bits only, since off-Heap addresses are absolute addresses it is possible that offset for that mode might have more
than 51 bits. Also a page size can never be more than 17GB. So to deal with this problem, spark subtracts the base offset
value from cursor/offset and use this value as relative offset for encoding. While decoding the record address reverse of
this operation is performed for off-Heap mode.

Next we will understand how [Task Memory Manager](TaskMemoryManager.md) handles the memory request and how its keeps 
track of all the allocated pages and some other important stuffs. 
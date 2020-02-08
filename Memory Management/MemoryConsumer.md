Talk about memory block, memory consumer, memory blocks. How encoding is done, how records are accessed, 
how they are stored. What all things a memory consumer keeps etc.
### Memory Location and Memory Block

In case of Off-Heap memory we can directly address memory using 64-bit addresses. Same is not the case in JVM addressed
memory. To address memory we need to know which object it belongs to and it's offset within that object. This means 128 
bits (64 bit for base address and 64 for offset) are required for addressing memory in on-Heap mode. There is need to 
maintain consistency i.e. memory addresses should be of similar size in both the modes. 

From above discussions it's clear that we need a base object reference and an offset to address a memory location. And to 
address a contiguous block of memory we need size of that memory block along with the other two values. Spark has defined 
*MemoryLocation* and *MemoryBlock* which serves respectively as memory location and memory block. MemoryLocation contains
base object reference and offset. MemoryBlock extends MemoryLocation. It also contains pageNumber and size.

Note : All the memory requests are granted in terms of pages only in spark. These pages are nothing but MemoryBlock.
Due to this reason only we have page numbers for memory blocks.

### Memory Consumers

Memory consumers are the one's who makes request for memory. They consume memory for performing various shuffle, sort,
hashing operations. Also they only deal with the tungsten memory(It will be discussed in detail later).
Some of the memory consumers





So how Exactly a memory consumer gets MemoryBlock/page ? Whenever a memory consumer wants to store some intermediate record
it first check whether it has any page with sufficient memory available. If its available then it will simply write the
record in that page. Otherwise, it will call *allocatePage()* of TaskMemoryManager. If its possible then TaskMemoryManager
will allocate a page to this memory consumer and stores this allocated memory block reference in page table's pageNumber 
index and finally returns that memory Block to consumer. After acquiring a page, memory consumer will update its own list
of pages and set all cursor to zero for this new page. Record will get written in this new page and all the other variables
value will be updated accordingly. To get encoded address this record memory consumer will call TaskMemoryManager's 
*encodePageNumberAndOffset()*. encodePageNumberAndOffset takes current page and cursor as parameter and returns the encoded
64-bit record address.

So How encoding is performed by TaskMemoryManager ?
This encoder takes the pageNumber and shifts it 51 bits in left. Mask the cursor with a 51-bit bitmask and then perform 
bitwise OR of these two values. Resultant value will be returned as encoded address. In case of Off-Heap Mode though we 
have make some considerations. As its clear from above discussion that offset is of 51 bits only, since off-Heap addresses
are absolute addresses it is possible that offset for that mode might have more than 51 bits. Also a page size can never be
more than 17GB. So to deal with this problem, spark subtracts the base offset value from cursor/offset and use this value 
as relative offset for encoding.
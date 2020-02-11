# WIP - Contents will be updated soon


# Task Memory Manager
Task Memory Manager 


*Paging*

To solve the above problem spark encodes the memory addresses into 64 bits for both the modes using following approach:
Use 64 bits addresses as absolute addresses in off-Heap mode. While in on-heap mode use upper 13 bits for page number and 
remaining 51 bits as offset within the page. These page numbers will act as indexes in Page Table which stores base memory 
addresses. TaskMemoryManager has this page table and it manages all the pages allocated to a particular task.

Since we have fixed 13 bits for page number, that means we can address 8192 pages. So that's the maximum size of page table.
Maximum size of a page is around 17GB. Thus, we can address around 140 GB of memory for a task.

Now let us understand how encoding is actually done :
a.) On-Heap Mode- 
Use images for explaining how encoding is done in both modes and also write some lines about it.




*Page Allocation*

PageTable is nothing but a MemoryBlock itself. PageTable is defined as an array of MemoryBlocks. All the memory allocations
in spark is done in terms of pages only. Page is nothing but a MemoryBlock. Page size in spark is not fixed. This is done
so that a record, however big, gets stored in single page only. 
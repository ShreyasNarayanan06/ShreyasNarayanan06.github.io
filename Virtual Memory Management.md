## Virtual Memory Structure

- Virtual Memory Management is primarily built upon the idea of page tables
    
- This necessitates that there be a page table structure. In this case and how most contemporary Operating Systems handle it is:
    
    - Making a Page Directory (1st level page table) which is allocated 1 page
    - Each entry in the PD holds the address of a page table
    - Each page table holds 1 pages worth (1024) of addresses to physical pages [[Physical Memory Management]]
    - Generally, each entry in the PD is set to 0 on initialization and then uses the physical page allocation to allocate a new page for each page table (the same holds true for how pages within the page table are allocated)
    - **This derives the following hierarchy: Page Directory → Page Table → Page**

​

```c
void vmem_init() {
    uint32_t page_directory_addr = (uint32_t)pm_alloc_page(); //allocates a page for the PD
    page_directory = (uint32_t*)page_directory_addr; //sets the PD pointer globally
    
    // Set all of the bits in the top level page table to 0
    for (int i = 0; i &lt; 1024; i++) {
        page_directory[i] = 0;
    }
    
    uint32_t page_table_1_addr = (uint32_t)pm_alloc_page(); //get the address of a new page table
    uint32_t* page_table_1 = (uint32_t*)page_table_1_addr; //set the pointer
    
    for (int i = 0; i &lt; 1024;  //Same as before
        page_table_1[i] = 0;
    }
    
    // Set the page directory index 0 to the pt1 we made
    page_directory[0] = (page_table_1_addr | 0x3);
    
    page_directory[1023] = (uint32_t)page_directory | 0x3; //Turn on all relevant setting for the page table recursive entry
    
    set_page_directory(page_directory_addr);  // Set the page directory
    enable_paging();  // Enable paging
}
```

- One quirky implementation detail from `vmem_init()` is that we set the final entry of the page table to be equal to the address of the page table. This is so that we have a fixed virtual address from which we can edit physical page tables from. This is covered in depth more with `kmalloc()` and `kfree()`.

# Accessing Virtual Memory

- The process of accessing virtual memory is fairly simple but uses some built in features from the CPU to make it happen
    
- A virtual memory address can be divided into 3 separate parts: Page Directory Index | Page Table Index | Page Offset all which are self explanatory
    
- When a certain virtual address is accessed the following is what happens under the hood. This operation is performed by a hardware component: the MMU:
    
    1. The CPU uses the `%cr3` register, which is set in `vmem_init()` to find the physical address of the page directory (assembly shown below)
    2. The CPU uses the PD Index to find the correct page table
    3. The CPU follows the pointer to that address and uses the page table index to find the correct page
    4. The CPU follows the address to the correct physical page and uses the offset to get the requested data

The relevant assembly functions are used in conjunction with the infrastructure set up in `vmem_init()` in order to allow us to access physical memory using virtual memory addresses:

```x86asm
get_page_directory:
    mov %cr3, %eax
    ret

set_page_directory:
    mov 4(%esp), %eax
    mov %eax, %cr3
    ret

enable_paging:
    mov %cr0, %eax
    or %eax, 0x80000000
    mov %eax, %cr0
    ret
```


# Allocating Virtual Memory

- Allocating virtual memory is primarily done through using `kmalloc()` and `kfree()`
- Both of these functions require some overhead to be setup beforehand in order for them to work as seamlessly as possible
- This overhead comes in the form of the heap, which is a region of memory used for dynamic allocation
	- This allows us to request n byte sized blocks in memory at runtime without having to set aside a massive portion of memory before compile
	- The way this works is that we allocate a `heap_start` and `heap_end` which are both virtual memory addresses and set a next_free pointer equal to `heap_start`
	- This means that we can track the next free page in memory within the heap which makes allocation a fairly straightforward process
	- Each virtual block has a header allocated before it which lists out the blocks size and free status. This allows us to make a linked list in the header where 1 header points to the next
	- This is quite a rudimentary process, however, because if a `kfree()` happens on a piece of data, its impossible to utilize that data afterwards because the linked list grows into empty, never allocated, space on the heap
	- This will get replaced by a more complex [[Bitmap Heap Manager]]

```c
void heap_init() {
	heap_start = (uint32_t*)0xD0000000; //arbitrary start, since we're in vmem for the heap it does not really matter where it starts only that it doesnt collide with other vmem addresses
	
	heap_end = (uint32_t*)0xE0000000;
	next_free = heap_start;
}
```

​Below is the `kmalloc()` function which uses a number of the principles listed above
```c
uint32_t* kmalloc(uint32_t size) { //fix approach later. needs to not be a bump allocator
	struct heap_header* header = (struct heap_header*)next_free;
	header->is_free = 0;
	header->size = size;
	header->prev = prev_free;
	header->next = NULL;
	prev_free = header; //prev_free is a global variable which tracks the prev_free header
	
	uint32_t addr = (uint32_t)next_free + sizeof(struct heap_header) + size;
	
	if (addr % 8 != 0) { //aligns the newly allocated memory to 8 byte boundary
		addr += (8 - (addr % 8));
	}
	
	next_free = (uint32_t*)addr;
	
	if (next_free > heap_end) { //handles when we run out of memory
		terminal_writestring("Out of memory\n");
		while(1);
	}
	
	return (uint32_t*)(header + 1);
}
```

### The Fault Handler

- The fault handler is a function which deals with when the kernel tries to access a value in memory that has not yet been allocated
- The first step in the fault handlers operation is getting the address of the fault from the CPU. 
	- The MMU discussed before, if there isn't a mapped address, will put the faulting_address into the `%cr2` register
	- We can read this with the function below
	
```x86asm
read_faulting_address:
	mov %cr2, %eax
	ret
```

- From the address we can get the page directory index, page table index, and the page offset
- This allows us to create a page in the correct place in memory (where the faulting address was)
- We make sure to 0 out any data in the new page just for safety as well
- Finally, we `iret` to return from the interrupt that generated the faulting address and allow us to re-enter the main operation loop
- We also flush the TLB to remove any bogus values

The code for the `fault_handler()` is below:

```c
void fault_handler() {
	uint32_t fault_addr = read_faulting_address(); //get the faulting addr from asm
	
	uint32_t pdidx = fault_addr >> 22; //extract the correct page directory idx
	uint32_t ptidx = (fault_addr >> 12) & 0x3FF; //same for page table
	
	if((virtual_PD[pdidx] & 0x1) == 0) { // if the value in the page table entry is 0 then we know its not been allocated so we need to allocate a new page
		virtual_PD[pdidx] = pm_alloc_page() | 0x3; //set all the variables in the entry
		for(int i = 0; i < 1024; i++) virtual_PTs(pdidx)[i] = 0; // 0 out all the data
	}
	
	  
	
	if((virtual_PTs(pdidx)[ptidx] & 0x1) == 0) { //create a new page in the pt if needed
		virtual_PTs(pdidx)[ptidx] = pm_alloc_page() | 0x3; 
	}
	
	flush_TLB(fault_addr); //flush TLB
	asm volatile ("iret"); //IRET
}
```

- **Recursive Mapping**: By setting the last entry of the Page Directory to its own physical address, the MMU "tricks" the CPU into treating the Page Directory as a Page Table, providing a fixed virtual address where all your Page Tables are mapped and accessible.
- **Avoiding the Catch-22**: This `virtual_PD` is necessary because while paging is active, the kernel cannot directly access physical addresses; recursive mapping allows you to edit Page Table entries using standard pointers without manually mapping and unmapping physical frames every time a fault occurs.
### `kfree()`

- The `kfree()` function works by simply combining free blocks in memory
- This requires just making sure the headers are representing the correct size in memory and pointing to the next header and previous header properly

The code for `kfree()` is below:

```c
void kfree(uint32_t* data_ptr) {
	struct heap_header* header = (struct heap_header*)data_ptr - 1;
	header->is_free = 1;
	
	if(header->next != NULL && header->next->is_free) {
		header->size += header->next->size + sizeof(struct heap_header);
		header->next = header->next->next;
		if(header->next) header->next->prev = header;
	}
}
```

## Virtual Memory Structure

- Virtual Memory Management is primarily built upon the idea of page tables
    
- This necessitates that there be a page table structure. In this case and how most contemporary Operating Systems handle it is:
    
    - Making a Page Directory (1st level page table) which is allocated 1 page
    - Each entry in the PD holds the address of a page table
    - Each page table holds 1 pages worth (1024) of addresses to physical pages
    - Generally, each entry in the PD is set to 0 on initialization and then uses the physical page allocation to allocate a new page for each page table (the same holds true for how pages within the page table are allocated)
    - **This derives the following heirarchy: Page Directory → Page Table → Page**

​

```c
void vmem_init() {
    uint32_t page_directory_addr = (uint32_t)pm_alloc_page(); //allocates a page for the PD
    page_directory = (uint32_t*)page_directory_addr; //sets the PD pointer gloablly
    
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

- One quirky implementation detail from vmem_init() is that we set the final entry of the page table to be equal to the address of the page table. This is so that we have a fixed virtual address from which we can edit physical page tables from. This is covered in depth more with kmalloc and kfree.

# Accessing Virtual Memory

- The process of accessing virtual memory is fairly simple but uses some built in features from the CPU to make it happen
    
- A virtual memory address can be divided into 3 seperate parts: Page Directory Index | Page Table Index | Page Offset all which are self explanatory
    
- When a certain virtual address is accessed the following is what happens under the hood:
    
    1. The CPU uses the CR3 register, which is set in vmem_init() to find the physical address of the page directory (assembly shown below)
    2. The CPU uses the PD Index to find the correct page table
    3. The CPU follows the pointer to that address and uses the page table index to find the correct page
    4. The CPU follows the address to the correct physicl page and uses the offset to get the requested data

The relevant assembly functions are used in conjunction with the infrastructure set up in vmem_init() in order to allow us to access physical memory using virtual memory addresses:

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

​

​
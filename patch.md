# Cache-based checkpointing mechanism for High-Availability in Safety Critical Systems
<!--Our paper regarding this topic is in this repo, you can find it [here](https://github.com/AntonioSposito/bao-progetto/blob/main/A2T%20paper.pdf).-->

What if we wanted to introduce in modern hypervisors a fast cache-based checkpointing mechanism with a low latency and overhead in order to reduce the downtime of the system? Our work focused on studying the feasibility of this mechanism exploiting the cache space efficiently

What we did:
- We modified Bao code to support a page permission restoration mechanism in memory.
- To optimize the access to the same page in the future, we have also explored the feasibility of pre-fetching the page within the restoration mechanism.

<!--
## Objectives
During the initialization phase, Bao assigns read and write permissions (and other flags) to each page, allowing VMs to access memory for any operation. We caused a data abort exception fault by setting the VM permissions for a page to write only, after the VM attempts a read operation the exception is raised and the hypervisor has to intervene. Since Bao does not support an handler for this fault, we patched the code by adding a simple handler that calculates the physical address of the page, finds the related page table entry, and restores normal read and write permissions.
To determine the impact of such operation, we measured the time required to execute the VM exit, the time needed to execute the handler we implemented and, finally, the time needed to execute the VM entry; then we compared this data with the time required to do a memory read operation from both disk and cache. By comparing these times, we were able to quantify the impact such a mechanism would have if it was implemented in Bao.
To optimize subsequent memory accesses to the page that caused the fault a pre-fetching of the page could be implemented within the handler itself; unfortunately, we have not been able to add this feature to our handler. -->

## How to use
You can simply follow the steps reported below and modify the code, or you can download it from [here](https://www.mediafire.com/file/ah7v943y2auico1/bao-demos3.zip/file). By downloading the zip file, after modifing some paths and user name in the .sh files, you can simply launch qemu as explained in the `automake.md` file.

## Our Patch
In order to understand the code of our patch we will explain first how Bao assigns page permissions, second, we will illustrate the baremetal guest code executed the VM managed by the hypervisor and finally we will present the code of the handler that restore access permissions.

### mem_map function
Bao uses a function called `mem_map` (located in the `\src\bao\core\mem.c` file) to map the allocated physical pages with the corresponding virtual pages filling in this way the translation tables.

Within this function, after several checks, the `pte_set` function is called to set the Page Table Entries (PTEs) at each of the 3 translation level; we are interested in a single PTE which refers to a page at translation level 3. The `pte_set` function makes simply an OR between the address of the page address `addr`, the page type `type`, and the access permission flags `flags` and saves this information in the address pointed by `pte`.

We  modified the function: when the physical address (0x401a7000) of the page we want to use in the experiment is encountered only write permissions are assigned to it, while the default permissions (write and read) are set for the other pages.

```c
int mem_map(...){
[...]
if (paddr == 0x401a7000){
	uint64_t wo_flags=PTE_VM_FLAGS_WO;
	pte_set(pte, paddr, pt_pte_type(&as->pt, lvl), wo_flags);
}else{
	pte_set(pte, paddr, pt_pte_type(&as->pt, lvl), flags);
}
[...]
}
```

The flags we used are defined in the file `srcs\bao\src\arch\armv8\inc\arch\page_table.h\`


## VM Baremetal Code
We modified the baremetal bao demo, the code run by the VM is located into the c file called `srcs\baremetal\src\main.c`. We added a write operation into the page (we use the IPA, not the physical address in the main) we used for our experiment. This operation is the only one allowed by the previously set permissions. 

```c
int* ptrA = NULL;
void main (void){
[...]
    ptrA = 0x1a2000;
    *ptrA=5;
[...]
}    
```
The code also stes up an UART interrupt, we can use the receiving handler `uart_rx_handler` to implement the reading operation that will trigger the data abort fault.

```c
void uart_rx_handler(){
static int counter = 0;
printf("\n\ncpu%d: %s: %d\n", get_cpuid(), __func__, counter++);
int a = 0;		
int ptr = 0x1a2000;	
 
MSR(PMCCNTR_EL0, 0);	

uint64_t begin = MRS(PMCCNTR_EL0);
asm volatile("ldr %0, [%1] ": "=r"(a): "r" (ptr): "memory"); //read
uint64_t bar_bef =  MRS(PMCCNTR_EL0);
asm volatile("dsb sy" ::: "memory");		
asm volatile("isb" ::: "memory");		
uint64_t bar_aft =  MRS(PMCCNTR_EL0);
printf("a = %x \n", a);	
uint64_t end = MRS(PMCCNTR_EL0);

[...]
uart_clear_rxirq();}
```

After printing the number of the CPU that handles the signal, a **LDR** (Load with immediate offset) operation is used to read from the page we are using. Since the page has been initialized without read permissions, a data abort fault is raised and the hypervisor has to intervene to handle it. After the LDR operation we placed a Memory Synchronization Barrier (the program waits for all operations within the memory to be completed before proceeding) and an Instruction Synchronization Barrier (no other instructions are pre-fetched and the processor's pipeline is flushed).  
The **MSR** instruction (Move to system co-processor register from ARM register) is used to reset the **PMCCNTR_EL0** register which holds the value of the Cycle Counter, **CCNT**. Between operations we read this register with the **MRS** instruction (system co-processor register to ARM register). We will use this information to calculate the number of cycles (and the time) required for each operation. 

### PMU
In the main file, we added the function `setup_PMU()` to set up the Performance Monitoring Unit, which we used to enable the clock counter

## Permission handler
The first time the LDR operation is performed, the hypervisor steps in to handle the exception. Since Bao does not foresee the possibility that a page can be mapped without read or write permissions, a handler is not implemented in the code to handle our case. We patched the code so that when the syndrome corresponding to the exception is encountered, a handler wrote by us is executed, instead of the error message saying that the hypervisor has no way to handle the exception.

This handler is located in `srcs\bao\src\arch\armv8\aborts.c`
```c
void permission_handler(uint32_t iss, uint64_t far, uint64_t il){

uint64_t beg_handler=MRS(PMCCNTR_EL0);

int paddr = (int)far - (int)0x14000 + (int)0x40019000;
pte_t* pte = NULL;
pte=pt_get_pte(&cpu.vcpu->vm->as.pt,  0x2, (void*)far);

pte_set(pte, (uint64_t)paddr, pt_pte_type(&cpu.vcpu->vm->as.pt,0x2),  PTE_VM_FLAGS);
	
uint64_t end_handler=MRS(PMCCNTR_EL0);
}
```

The handler is quite simple. In input it receives a 32-bit integer **ISS** (Instruction Specific Syndrome), a 64-bit integer **FAR** which holds the faulting address and a 64-bit integer **IL** which holds the Instruction Length for the current exception. First we calculate the physical address (paddr) of the page starting from the FAR using a base and an offset; then we get the pointer to the page table entry we are trying to modify using the function `pt_get_pte`, as input values we use the address of the page table, the level of the multi-level translation and the faulting address. After getting a pointer to the PTE, we can give all the access permissions to the page with the function `pte_set` that makes an OR between the physical address, the page type and the flags. 

After using the `pte_set` function, we could add an LDR operation to bring the page into the cache and speed up future memory reads and to better control the cached data (achieving in this way a chache based checkpointing system). We want to do this because the page will have all the necessary permissions, which means it can be read directly from the cache instead of having to go to memory first. This will make the process faster and more efficient. However, when we tried to test this read operation, we were unsuccessful and didn't have enough time to investigate the problem. We think that caching the page in our handler managed by the hypervisor caused some memory misalignment, and during execution, the CPU encountered an error and stopped.

In order to execute our handler, we need to modify the function `aborts_sync_handler` also located in `srcs\bao\src\arch\armv8\aborts.c`, when the ISS corresponding to our fault is encoutered, the permission handler we wrote is executed:

```c
void aborts_sync_handler(){
[...]
    if (iss == 0x1c1800f){
	permission_handler(iss, ipa_fault_addr, il);
    }else{
	if (handler)
        	handler(iss, ipa_fault_addr, il);
    	else
        	ERROR("no handler for abort ec = 0x%x", ec);  // unknown guest exception
    }
}
```

<!--## Results and conclusions -->
<!--You can read our results and conclusions in [our paper](https://github.com/AntonioSposito/bao-progetto/blob/main/A2T%20paper.pdf) -->

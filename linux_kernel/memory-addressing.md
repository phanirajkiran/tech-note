Memory Addressing
=================

# Memory Address

- *Logical address*
  - Included in the machine language instructions to specify the address of an operand or of an instruction. Each logical address consists of a *segment* and an *offset* (or *displacement*) that denotes the distance from the start of the segment to the actual address.
- *Linear address* (a.k.a. *virtual address*)
  - A single 32-bit unsigned integer that can be used to address up to 4 GB. Linear addresses are usually represented in hexadecimal notation; their values range from **0x00000000 to 0xffffffff**.
- *Physical address*
  - Used to address memory cells in memory chips. Physical addresses are represented as 32-bit or 36-bit unsigned integers.

The Memory Management Unit (MMU) transforms a logical address into a linear address by means of a hardware circuit called a segmentation unit; subsequently, a second hardware circuit called a paging unit transforms the linear address into a physical address.

![Logical address translation](http://i.imgur.com/cFFwLoe.png)

# Segmentation in Hardware

Starting with the 80286 model, Intel microprocessors perform address translation in two different ways called *real mode* and *protected mode*. Real mode exists mostly to maintain processor compatibility with older models and to allow the operating system to bootstrap.

## Segment Selectors

A logical address consists of two parts: a segment identifier and an offset that specifies the relative address within the segment. The segment identifier is a **16-bit** field called the *Segment Selector*, while the offset is a **32-bit** field.

- 15~3 *index*: Identifies the Segment Descriptor entry contained in the GDT or in the LDT
- 2~1 *TI*: Table Indicator, 0 for GDT, 1 for LDT
- 0 *RPL*: Requestor Privilege Level, specifies the Current Privilege Level of the CPU when the corresponding Segment Selector is loaded into the cs register; it also may be used to selectively weaken the processor privilege level when accessing data segments

## Segmentation Registers

To make it easy to retrieve segment selectors quickly, the processor provides segmentation registers whose only purpose is to hold Segment Selectors; these registers are called *cs*, *ss*, *ds*, *es*, *fs*, and *gs*.

- *cs*: The **code segment** register, which points to a segment containing program instructions
- *ss*: The **stack segment** register, which points to a segment containing the current program stack
- *ds*: The **data segment** register, which points to a segment containing global and static data

The remaining three segmentation registers are general purpose and may refer to arbitrary data segments.

The *cs* register has another important function: it includes a 2-bit field that specifies the Current Privilege Level (CPL) of the CPU. The value 0 denotes the highest privilege level, while the value 3 denotes the lowest one. Linux uses only levels 0 and 3, which are respectively called Kernel Mode and User Mode.

## Segment Descriptors

Each segment is represented by an 8-byte *Segment Descriptor* that describes the segment characteristics. Segment Descriptors are stored either in the *Global Descriptor Table (GDT)* or in the *Local Descriptor Table (LDT)*.

Usually only one GDT is defined, while each process is permitted to have its own LDT if it needs to create additional segments besides those stored in the GDT. The address and size of the GDT in main memory are contained in the *gdtr* control register, while the address and size of the currently used LDT are contained in the *ldtr* control register.

- *Code Segment Descriptor*
  - either in the GDT or in the LDT
  - *S* flag set (non-system segment)
- *Data Segment Descriptor*
  - either in the GDT or in the LDT
  - *S* flag set
  - Stack segments are implemented by means of generic data segments
- *Task State Segment Descriptor (TSSD)*
  - only in the GDT
  - *S* flag is set to 0
- *Local Descriptor Table Descriptor (LDTD)*
  - only in the GDT
  - *S* flag is set to 0

## Fast Access to Segment Descriptors

The 80x86 processor provides an additional nonprogrammable register for each of the six programmable segmentation registers. Every time a Segment Selector is loaded in a segmentation register, the corresponding Segment Descriptor is loaded from memory into the matching nonprogrammable CPU register. rom then on, translations of logical addresses referring to that segment can be performed without accessing the GDT or LDT stored in main memory.

![Segment Selector and Segement Descriptor](http://i.imgur.com/FI9J76P.png)

## Segmentation Unit

- Examines the *TI* field of the Segment Selector to determine which Descriptor Table stores the Segment Descriptor (GDT or LDT).
- Computes the address of the Segment Descriptor from the *index* field of the Seg-ment Selector. The *index* field is multiplied by 8 (the size of a Segment Descriptor), and the result is added to the content of the *gdtr* or *ldtr* register.
- Adds the offset of the logical address to the *Base* field of the Segment Descriptor, thus obtaining the linear address.

![Translating a logical address](http://i.imgur.com/0e9ucnK.png)

# Segmentation in Linux

Linux uses segmentation in a very limited way because segmentation and paging are somewhat redundant. Linux prefers paging to segmentation for the following reasons:

- Memory management is simpler when all processes use the same segment register values — that is, when they share the **same set of linear addresses**.
- Portability to a wide range of architectures

All Linux processes running in User Mode use the same pair of segments to address instructions and data. These segments are called *user code segment* and *user data segment*, respectively. Similarly, there are *kernel code segment* and *kernel data segment* for Kernel Mode.

The corresponding Segment Selectors are defined by the macros `__USER_CS`, `__USER_DS`, `__KERNEL_CS`, and `__KERNEL_DS` in `include/asm-x86_64/segment.h`.

Notice that the linear addresses associated with such segments all start at 0 and reach the addressing limit of 2^32 –1. This means that all processes, either in User Mode or in Kernel Mode, may use the same logical addresses.

An important consequence of having all segments start at `0x00000000` is that in Linux, **logical addresses coincide with linear addresses**.

The following snippet in `include/asm-x86_64/processor.h` shows the fact that all the processes in User Mode use single pair of segment:

```c
#define start_thread(regs,new_rip,new_rsp) do { \
    asm volatile("movl %0,%%fs; movl %0,%%es; movl %0,%%ds": :"r" (0));  \
    load_gs_index(0);                           \
    (regs)->rip = (new_rip);                         \
    (regs)->rsp = (new_rsp);                         \
    write_pda(oldrsp, (new_rsp));                        \
    (regs)->cs = __USER_CS;                          \
    (regs)->ss = __USER_DS;                          \
    (regs)->eflags = 0x200;                          \
    set_fs(USER_DS);                             \
} while(0)
```

For LDT, since Linux User Mode applications do not make use of it, the kernel defines a default LDT to be shared by most processes.

# Paging in Hardware

The paging unit thinks of all RAM as partitioned into fixed-length *page frames* (sometimes referred to as *physical pages*). Each page frame contains a page — that is, the length of a page frame coincides with that of a page. A page frame is a constituent of main memory, and hence it is a storage area. It is important to distinguish a page from a page frame; the former is just a block of data, which may be stored in any page frame or on disk.

The data structures that map linear to physical addresses are called *page tables*; they are stored in main memory and must be properly initialized by the kernel before enabling the paging unit.

## Regular Paging

Starting with the 80386, the paging unit of Intel processors handles **4 KB** pages.

The 32 bits of a linear address are divided into three fields:

- *Directory*: The most significant 10 bits
- *Table*: The intermediate 10 bits
- *Offset*: The least significant 12 bits

The entries of Page Directories and Page Tables contains **20 most siginificant bits** of a page frame physical address and **12 flag bits**.

The physical address of the Page Directory in use is stored in a control register named *cr3*. The Directory field within the linear address determines the entry in the Page Directory that points to the proper Page Table. The address’s Table field, in turn, determines the entry in the Page Table that contains the physical address of the page frame containing the page. The Offset field determines the relative position within the page frame. Because it is 12 bits long, each page consists of 4096 bytes of data.

If the `Present` flag of the entry of the Page Table is cleared, the page is not present in main memory; in this case, the paging unit issues a **Page Fault exception** while translating the linear address.

Both the Directory and the Table fields are 10 bits long, so Page Directories and Page Tables can include up to 1,024 entries. It follows that a Page Directory can address up to 1024 × 1024 × 4096=**2^32** memory cells, as you’d expect in 32-bit addresses.

![Paging by 80x86 processors](http://i.imgur.com/tshkQuw.png)

## Extended Paging

Starting with the Pentium model, 80×86 microprocessors introduce extended paging, which allows page frames to be **4 MB** instead of 4 KB in size. Extended paging is used to translate large contiguous linear address ranges into corresponding physical ones; in these cases, the kernel can do without intermediate Page Tables and thus save memory and preserve TLB entries.

In this case, the paging unit divides the 32 bits of a linear address into two fields:

- *Directory*: The most significant 10 bits
- *Offset*: The least significant 22 bits

![Extended paging](http://i.imgur.com/6VJ9lCq.png)

## Hardware Protection Scheme

While 80×86 processors allow four possible privilege levels to a segment, only two privilege levels are associated with pages and Page Tables.

- `User/Supervisor` flag
  - 0, can be addressed only when the CPL is less than 3 (Kernel Mode)
  - 1, can always be addressed
- `Read/Write` flag
  - 0, read-only
  - 1, can be read and written

## The Physical Address Extension (PAE) Paging Mechanism

PAE is activated by setting the Physical Address Extension (PAE) flag in the *cr4* control register. The Page Size (`PS`) flag in the page directory entry enables large page sizes (2 MB when PAE is enabled).

- The 64 GB of RAM are split into 2^24 distinct page frames, and the physical address field of Page Table entries has been expanded from 20 to 24 bits. Because a PAE Page Table entry must include the 12 flag bits and the 24 physical address bits, for a grand total of 36, the Page Table entry size has been doubled from 32 bits to 64 bits. As a result, a 4-KB PAE Page Table includes 512 entries instead of 1,024.
- A new level of Page Table called the *Page Directory Pointer Table (PDPT)* consisting of four 64-bit entries has been introduced.
- The *cr3* control register contains a **27-bit** Page Directory Pointer Table base address field. Because PDPTs are stored in the first 4 GB of RAM and aligned to a multiple of 32 bytes (2^5), 27 bits are sufficient to represent the base address of such tables.

When mapping linear addresses to 4 KB pages (`PS` flag cleared in Page Directory entry), the 32 bits of a linear address are interpreted in the following way:

- *cr3*: Points to a PDPT
- bits 31~30: Point to 1 of 4 possible entries in PDPT
- bits 29~21: Point to 1 of 512 possible entries in Page Directory
- bits 20~12: Point to 1 of 512 possible entries in Page Table
- bits 11~0: Offset of 4-KB page

However, the main problem with PAE is that linear addresses are still 32 bits long. This forces kernel programmers to reuse the same linear addresses to map different areas of RAM.

# Paging in Linux

Linux adopts a common paging model that fits both 32-bit and 64-bit architectures. Starting with version 2.6.11, a four-level paging model has been adopted:

- Page Global Directory
- Page Upper Directory
- Page Middle Directory
- Page Table

![The Linux paging model](http://i.imgur.com/3dedh53.png)

For 32-bit architectures with no Physical Address Extension, two paging levels are sufficient. Linux essentially eliminates the Page Upper Directory and the Page Middle Directory fields by saying that they contain zero bits.

For 32-bit architectures with the Physical Address Extension enabled, three paging levels are used. The Linux’s Page Global Directory corresponds to the 80×86’s Page Directory Pointer Table, the Page Upper Directory is eliminated, the Page Middle Directory corresponds to the 80×86’s Page Directory, and the Linux’s Page Table corresponds to the 80×86’s Page Table.

For 64-bit architectures three or four levels of paging are used depending on the linear address bit splitting performed by the hardware.

## Page Table Handling

Most of functions for page table handling are defined in `include/asm-<arch>/pgtable.h` and `include/asm-<arch>/pgalloc.h`.

## Physical Memory Layout

As a general rule, the Linux kernel is installed in RAM starting from the physical address 0x00100000 — i.e., from the second megabyte. The reason that kernel isn't loaded starting with the first availalbe megabyte of RAM includes:

- Page frame 0 is used by BIOS to store the system hardware configuration detected during the *Power-On Self-Test (POST)*; the BIOS of many laptops, moreover, writes data on this page frame even after the system is initialized.
- Physical addresses ranging from 0x000a0000 to 0x000fffff are usually reserved to BIOS routines and to map the internal memory of ISA graphics cards.
- Additional page frames within the first megabyte may be reserved by specific computer models.

## Process Page Tables

The linear address space of a process is divided into two parts:

- Linear addresses from `0x00000000` to `0xbfffffff` can be addressed when the pro- cess runs in either User or Kernel Mode.
- Linear addresses from `0xc0000000` to `0xffffffff` can be addressed only when the process runs in Kernel Mode.

The `PAGE_OFFSET` macro yields the value 0xc0000000; this is the offset in the linear address space of a process where the kernel lives.

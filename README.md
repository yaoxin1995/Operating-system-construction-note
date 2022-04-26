# 2.week

Class: HIST 230
Created: April 18, 2022 8:41 PM
Materials: https://www.theatlantic.com/international/archive/2014/12/the-real-story-of-how-america-became-an-economic-superpower/384034/
Reviewed: Yes
Type: Section

# Lecture

## Compilation / Linking

1. Why hello world can’t run on bare metal?
    - No dynamic linker available on bare metal (solution: link all necessary libraries statically)
    - Standard library relay on system calls (- > Can’t use Standard CPP library in OS development )
    - All functions and variable use virtual memory address, which is defined by linker  (- >need a custom linker config)
    - High-level language code needs environment setup (stack, CPU-register usage, address mapping)
        - Need own startup code (written in assembler) to setup the environment .
2. Is OS development in a high level programming language (C, CPP, etc.)?
    
    Yes, but can’t use standard libs.
    
3. Difference between static and shared linking
    - A **static library:  w**hen your program is linked against a static library, **the machine code of external functions used in your program is copied into the executable.**
        - Cons:
            - compile time is longer
            - output binary is bigger
    - A s**hared library:**  when your program is linked against a shared librar**y, only a small table is created in the executable. Before the executable starts running, the operating system loads the machine code needed for the external functions** .
        
        Pros:
        
        - **makes executable files smaller and saves disk space,** because one copy of a library can be shared between multiple programs. Furthermore,
        - **most operating systems allows one copy of a shared library in memory to be used by all running programs, thus, saving memory**.
        
        Cons:
        
        - start time is slower(linked content should be copied if the OS doesn’t loaded it yet)
        - linker errors are thrown in runtime

## Booting

Booting is a process of starting a computer. The process involves a chain of stages, in which at each stage, a smaller, simpler program loads and then executes the larger, more complicated program of the next stage:

- Start the system (turn on the machine)
- CPU jump to a fixed address, where has the BIOS code, and execute the BIOS code
- The BIOS performs basic system initialization (such as activating the video card and checking the amount of memory installed) and validation
- BIOS loads the operating system from some appropriate location such as  hard disk, CD-ROM, and passes control of the machine to the operating system

1. **The Boot Loader**
    
    Hard disks for PCs are divided into 512 byte regions called *sectors.* A sector is the disk's minimum transfer granularity: each read or write operation must be one or more sectors in size and aligned on a sector boundary. 
    
    If the disk is bootable, the first sector is called the *boot sector*, since this is where the boot loader code resides. When the BIOS finds a bootable floppy or hard disk, it loads the 512-byte boot sector (which store the boot loader code) into a fixed physical memory address, and then uses a jmp to jump to this location.  Then, CPU will execute the boot loader. 
    
    The boot loader must perform following things:
    
    - Define hardware/software state
        - Set up registers to switches the processor from real mode to *32-bit protected mode*
            - Real Mode
                
                In real mode, memory is limited to only one megabyte (2^20 bytes). Valid
                address range from (in hex) 00000 to FFFFF. These addresses require a
                20-bit number. Obviously, a 20-bit number will not fit into any of the
                8086’s 16-bit registers. Intel solved this problem, by using two 16-bit values
                determine an address. The first 16-bit value is called the selector. Selector
                values must be stored in segment registers. The second 16-bit value is called
                the offset. The physical address referenced by a 32-bit `selector:offset` pair is
                computed by the formula `16 ∗ selector + offset`
                
                Con:
                
                - A single selector value can only reference 64K of memory (the upper
                limit of the 16-bit offset). What if a program has more than 64K of
                code? A single value in CS can not be used for the entire execution
                of the program. The program must be split up into sections (called
                segments) less than 64K in size. When execution moves from one segment to another, the value of CS must be changed. Similar problems
                occur with large amounts of data and the DS register. This can be
                very awkward!
                - Each byte in memory does not have a unique segmented address. The
                physical address 04808 can be referenced by 047C:0048, 047D:0038,
                047E:0028 or 047B:0058. This can complicate the comparison of segmented addresses.
            - Protect mode
                
                Protected mode uses a technique called virtual memory
                
                1. 16-bit Protected Mode
                    - segments are moved between memory and disk as needed
                        
                        When a segment is returned to memory from disk, it is very likely that it will be put into a different area of memory that it was in before being moved to disk. All of this is done transparently by the operating system. The program does not have to be written differently for virtual memory to work.
                        
                    - each segment is assigned an entry in a descriptor table
                        
                        This entry has all the information that the system needs to know
                        about the segment. This information includes: is it currently in memory;
                        if in memory, where is it; access permissions (e.g., read-only). The index
                        of the entry of the segment is the selector value that is stored in segment
                        registers
                        
                    - Con: segment sizes are still limited to at most 64K, because the offset are still 16bit long
                2. 32-bit Protected Mode
                    
                    Two major differences between 386 32-bit and 286 16-bit protected modes:
                    
                    - Offsets are expanded to be 32-bits. This allows an offset to range up
                    to 4 billion. Thus, segments can have sizes up to 4 gigabytes.
                    - Segments can be divided into smaller 4K-sized units called pages. The
                    virtual memory system works with pages now instead of segments.
                    This means that only parts of segment may be in memory at any one
                    time.
        - Set up the protected-mode data segment registers, etc.
    - Load the kernel/system
    - Jump into loaded system

## Testing

1. Printf debugging 
    1. printf() often changes the debuggee’s behavior  -> Problem vanishes / changes symptoms
    
2. Emulators: emulate real hardware in software
    - Simplifies debugging
    - Shorter development cyclles
    - **Emulation is a special case of virtualization, which provides a virtual resource Y based on based on a resource X**

### Debugging

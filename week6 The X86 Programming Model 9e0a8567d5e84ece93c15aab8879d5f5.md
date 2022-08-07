# week6 The X86 Programming Model

Created: July 20, 2022 4:29 PM
Reviewed: No

## x86-64 Assembler Programming (E04 10-36)

1. What is a Register?
    
    Extremely fast, very small storage within the CPU that can (in x86-64 CPUs) store 64 bits
    
2. Memory
    
    Main memory: Functionally like a gigantic array of registers, selectively 8, 16, 32 or 64 bits “wide”
    
    - smallest addressable unit: Byte
    - memory cells numbered consecutively index →
    - accesses are several 100x slower than to registers
    
    **Q: big / little endian?**
    
    **Big Endian Byte Order:** The **most significant** byte (the "big end") of the data is placed at the byte with the lowest address. The rest of the data is placed in order in the next three bytes in memory.
    
    **Little Endian Byte Order:** The **least significant** byte (the "little end") of the data is placed at the byte with the lowest address. The rest of the data is placed in order in the next three bytes in memory.\
    
3. The Stack
    
    Temporary LIFO storage for values “as long as they are needed”, each function has a stack frame to store the local variables
    
    - Support 2 Operation: Push (rsp -= 8, *rsp = item) and Pop (item = *rsp, rsp+=8)
    - stack pointer (x86-64: rsp) points to the top of stack
    - Note that X86 stack grow from high address to low address
4. Functions
    - **Q: Advantage compared to goto**: Call from arbitrary location in your program, return/continue the calling program part
        - Jump instruction: Changes rip to target address (absolute, or rip-relative)
        - Function call: like a jump, plus saves the return address old rip value (plus instruction length) is saved on the stack
        - Function return: ret pops address from stack, jumps there (the Instruction right after the function call)
    - The function itself doesn’t need to know where it was called from, and where to return afterwards (this happens automatically –**Q how? force by calling convention**)
    - X86 64 bit calling convention
        
        ```cpp
        Example instruction	What it does
        pushl %eax        subl $8, %esp
        									movl %eax, (%esp)
        
        popl %eax	        movl (%esp), %eax
        									addl $8, %esp
        
        call 0x12345	    pushl %eip (*)
        									movl $0x12345, %eip (*)
        
        ret	              popl %eip (*)
        
        C function prologue:
        									movl %ebp, %esp
        									popl %ebp		
        
        C function epilogue:
        									movl %ebp, %esp
        									popl %ebp			
        
        ```
        
    - Big example
        
        ![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled.png)
        
        - Compiler will add function prologue and epilogue for each function when it transfer c function to assembly
    - Parameters: the first 6 in registers, further ones on the stack and Parameters on the stack must be removed again afterwards
    - Access to parameters within the function:  Simplified by using the base pointer `rbp`
        - Convention: Save rbp at the beginning of a function, set to rsp
            - access the 7th parameter via [rbp+16]
            - access the 8th parameter via [rbp+24] …
            - … independently from whether rsp was changed in the meantime
            - Note [rbp] points to the old rbp adress, [rbp + 8] points to the return address
            - Use rpb we can implement the back trace for debug porpose
            
            ![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%201.png)
            
5. Calling Assembler Functions for high level language
    - Use key word `GLOBAL`, we can expose a assembly function to linker, i.e., make the assembly function to be global
        
        ```cpp
        ; EXPORTED FUNCTIONS
        [GLOBAL toc_switch]
        [GLOBAL toc_go]
        toc_go: ...
        ```
        
    - C++ program can call the function by using the key word `extern "C”` to disable the cpp name mangling
        
        ```cpp
        extern "C" void toc_go(struct toc* regs);
        
        Purpose of extern c:
        1. tell liner we have a function called toc_go, which is defined in other file
        2. for toc_go, use c name mangling instead cpp name mangling
        ```
        
    - If we call a assembly function, the assembly function as caller have to save the non-volatile register if it touch them

## ****Basic Programming Model****

- 8086: Programming Model
    - 16-bit architecture, little endian
        
        Big-endian is an order in which the "big end" (most significant value in the sequence) is stored first, at the lowest storage address. Little-endian is an order in which the "little end" (least significant value in the sequence) is stored first.
        
    - 20-bit address bus
    - few register
        - Extremely fast, very small storage within the CPU that can (in x86-64 CPUs) store 64 bits
    - 123 instructions
        - non-orthogonal instruction set: not all instructions can be combined with any registers→hard compiler
    - Opcode lengths of 1 to 4 bytes
    - Segmented memory
- 8086: Register file
    
    ![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%202.png)
    
    - Each “general-purpose” register fulfills a specific purpose
- Segmented Memory
    
    ![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%203.png)
    
    - logical addresses on 8086 consist of segment selector (the value of a segment register) and offset (from a general-purpose register or the instruction)
    - How are segment selector and offset used to calculate addresses?
        
        Segment selector(16 bit)<< 4 + offset(16 bit) = Physical address(20 bit)
        
    - Assessment
        
        Pro:
        
        - allows the CPU to access the 20 bit address bus using the 16 bit registers
        - it allows programs to be loaded into any segment address without having to recalculate the individual addresses of variables – they are merely references as offsets from the segment.
        
        Con:
        
        - very complex
        - the largest memory segments size is 64 KB, therefore OS can’t run a program that is bigger than 64kb
        
- 8086: Memory Models
    
    Addressing can be done differently by programs. This resulted in different memory models
    
    - Tiny:
        - Code, data and stack segments are identical
        - 64 kiB in total
    - Small
        - Code separated from data & stack
        - 64 kiB + 64 kiB
    - Medium
        - 32- (or rather 20-) bit “far” pointers for code
        - 16 bit “near” pointers for data & stack (fixed 64 kiB segment)
    - Compact
        - 16-bit “near” pointers for code (fixed 64-kiB segment)
        - 32- (20-) bit “far” pointers for data & stack
    - Large
        - “far” pointers for everything – 1 MiB completely usable
    - Huge
        - like “large”, but with normalized pointers
    - Q What is the difference between "near", "far" and "huge" pointers?
        - Near pointers has 16 bit and refer (as an offset) to the current segment, i.e., it has implied selector
        - **A far pointer** is typically 32 bit (16bit selector + 16bit offest) that can access memory outside current segment.  To use this, compiler allocates a segment register to store segment address, then another register to store offset within current segment. Note 2 different far pointer may point to same location
        - **huge pointer** is also typically 32 bit and can access outside segment. And it is normalized to have the highest possible segment for a given address. For one location we only have 1 correspondent huge pointer

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%204.png)

- 8086: Conclusion
    - first PC’s CPU and still compatible
    - 1 MiB of memory in spite of 16-bit architecture(normally 20 bits), Segments (each is 64kb) separate logical modules in memory.
    - Difficult program and compiler development:
        - different memory models,
        - non-orthogonal instruction set
- IA-32: Intel’s 32-bit Architecture
    - 32-bit technology: Registers, data and address bus
        - starting with Pentium Pro: 64-bit data and 36-bit address bus
    - more registers
    - Complex support for protection and multitasking
        - Protected Mode(the ring)
        - originally introduced with the 80286 (16 bit)
    - Compatibility
    - Segment-based programming model
    - Page-based MMU
- amd64 / x86-64 – 64-Bit AMD/Intel Arch
    - IA-32 was limited to max. 4 GiB virtual memory
    - Solution: 64-bit architecture
        - memory address and register have 64 bit, but currently only the first 48-bits are used for virtual address space (up to 256 TiB)
    - memory protection in Page-based programming model
        - X(excuteable) bit for individual bit
        - nearly no segmentation (flat memory)
    - Still backwards compatible
        - Long Mode for putting the new features to use
        - Real Mode for older operating systems

## **Addressing modes**

An *addressing mode* is an expression that calculates an address in memory to be read/written to. These expressions are used as the source or destination for a `mov` instruction and other instructions that access memory.

- Common address mode
    
    ![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%205.png)
    
- x86-64 has additional support for calculating effective address
    - what is effective address
        
        **effective address is also referred to as offset address,**  1MB memory is divided into 16 segments. Each segment has their own offset address and the segment start **address is defined by the segment registers.** 
        
        linear address = effective address + segment start address
        
        physical address = linear address (if page table is not enabled) / linear address go through the page table (if page table enabled)
        
    - effective addresses = BASE-REG +(INDEX-REG * Scale) + Displacement
    - IP-relative addressing: Effective addresses := RIP + Displacement
- Read more about address mode
    
    ```cpp
    Operands can be immediate values, registers, or memory values.
    Immediates are specified by a $ followed by an integer in standard C notation. In nearly all cases, immediates are
    limited to 32 bits.
    For all but a few special instructions, memory addresses are specified as
    offset(base,index,scale)
    where base and index are registers, scale is a constant 1,2,4, or 8, and offset is a constant or symbolic
    label. The effective address corresponding to this specification is (base + index × scale + offset).. Any of the
    various fields may be omitted if not wanted; in effect, the omitted field contributes 0 to the effective address (except
    that scale defaults to 1). Most instructions (e.g., mov) permit at most one operand to be a memory value.
    Instructions are byte-aligned, with a variable number of bytes. The size of an instruction depends mostly on the
    complexity of its addressing mode. The performance tradeoff between using shorter, simpler instructions and longer,
    more powerful ones is complex.
    Offsets are limited to 32 bits. This means that only a 4GB window into the potential 64-bit address space can be
    accessed from a given base value. This is mainly an issue when accessing static global data. It is standard to access this
    data using PC-relative addressing (using %rip as the base). For example, we would write the address of a global
    value stored at location labeled a as a(%rip), meaning that the assembler and linker should cooperate to compute
    the offset of a from the ultimate location of the current instruction.
    ```
    

## The A20 Gate

On the IBM XT (8086), address calculation could overflow, therefore we use the A20 gate to wrap around the overflowed result, so that the result address is with 1MB. For accessing the address memory greater than 1 MiB in more advanced machine, the A20 gate must be explicitly enabled. (switch from real mode to long mode)

## IA-32: Protected Mode – Segments

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%206.png)

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%207.png)

## x86-64: Page-based MMU

- Demand paging: only load pages that are demanded by the excuting process.
- 80386: Paging Unit(PU) can be enabled optionally
- Configured via ctrl register CR0/2/3

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%208.png)

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%209.png)

## x86-64 :TLB

Page table based address translation is very expensive. Therefore CPU use the TLB to cache the virtual physical address pair. CPU first look at the cache, if it doesn’t find the result, it then start translate the virtual address to physical address based on page table. Note Writing CR3 (switch page table) invalidates the TLB.

## Protection on IA-32

### Protection Rings and Gates

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%2010.png)

- A 2-bit entry in the segment descriptor assigns each segment to a privilege level.
- Privilege level 3 is the lowest of the four levels and meant for applications.
- Privilege level 0 is the highest and reserved for the OS kernel
- Any violation triggers an exception

### Protection on x86-64

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%2011.png)

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%2012.png)

## IA-32: Multitasking

IA-32: Multitasking: segment for every task is seperated (certical and horizontal). 

### [Task-State-Segment](https://en.wikipedia.org/wiki/Task_state_segment)

- Task Reg(TR) points to TSS
- on IA-32: storage space for task state
    - segments, reg contents
    - task switches possible in hardware
- not in x86 long mode
    - instead: TSS is used for I/O permissions check and store stack pointer of the task in each privilege level
        - Access to devices in memory (memory-mapped I/O) controllable via memory protection
        - Access to I/O ports restricted by
            - I/O Privilege Level bits in RFLAGS I/O OK when on specified protection rings
            - Other rings: I/O Permission Bitmap in TSS controls access

## Summary

![Untitled](week6%20The%20X86%20Programming%20Model%209e0a8567d5e84ece93c15aab8879d5f5/Untitled%2013.png)

##
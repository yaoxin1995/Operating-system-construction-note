# exercise-3

- Course: OSC lab
- [Material](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/E03-Traps-startup.pdf)
- [video](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/E03-Traps-startup.mp4)
- [task-2](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/tasks/task2/)

# Traps

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled.png)

### Distinguish Trap and Interrupt

- Trap : triggerd by an instruction
    - trap or int instruction for syscalls
    - Undefined result(/0)
    - Hardware problem(bus error)
    - PF(need OS do sth)
    - Invaild instruction(programming error)
    - Porperities
        - normally predictable, often reproducible
        - Restart or abort the triggering activity
- INTR : triggerd by hardware
    - Hardware requires attention by software
    (Timer, Keyboard controller, Hard-disk controller, …)
    - Porperities
        - not predictable, not reproducible
        - resume the interrupted activity

## Traps – Overview

- Triggered by the CPU if it determines a **problem** executing the current instruction
    - Division by 0
    - Bus error (program generates invalid address)
    - OS must do something (e.g. page fault)
    - Invalid instruction
- or if the executed instruction is supposed to cause a trap
    - **int 0x80**
    - **syscall / sysenter**
    

## x86 IDT: Structure

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%201.png)

## Traps – Examples

### Division by zero

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%202.png)

- received signal : trap handler is in the kernel, once ur program casued trap the CPU switches to supervisior mode and the handler in the os handles this trap , the OS informs the usermode program that it caused the trap by forwarding this info to that programm through a signal SIGFPE
- Signal is actually some emulating traps and intr in user mode

### Page Fault

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%203.png)

- 0 is usually an address that no mapped in the os that use virtual mem
- in oostubs 0 is been mapped.

# Lab Task #2

## What does startup.asm do?

### x86-64 Interrupt Descriptor Table

- **max. 256 entries**
    - Base address and size in IDTR
- **16 bytes per entry (“gate”)**
    - Task gate (Hardware tasks)
    - Trap gate (Exception handler)
    - Interrupt gate (Exception handler + cli)
    - main difference of intr gate and intr gate
        - when entry is triggered…
            - intr gates lead to exception handler being run with diable intrs
            - trap handlers are run with intr enabled

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%204.png)

- the CPU has 1 register called **IDTR** which contains the base and the limit(size) of this intr descroptor table, and IDT resides in mem.
- IDT is called gate by intel

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%205.png)

- startup.asm
    - this happens at compile time, this generate the descriptor table in the object file.

```nasm

[SECTION .data]
; [...]
; Interrupt descriptor table with 256 entries
idt:
%macro idt_entry 1
	 dw (wrapper_%1 - wrapper_0) & 0xffff ; offset, start of our intr handler func
	 dw 0x0000 | 0x8 * 2 ; 64-bit code segment selector
	 dw 0x8e00 ; present + 80386 32-bit interrupt gate
	 dw ((wrapper_%1 - wrapper_0) & 0xffff0000) >> 16 ; offset
	 dd ((wrapper_%1 - wrapper_0) & 0xffffffff00000000) >> 32 ; offset
	 dd 0x00000000 ; reserved
%endmacro

%assign i 0
%rep 256
idt_entry i     
%assign i i+1
%endrep
idt_descr:
	 dw 256*8-1 ; 256 entries
	 dq idt
```

- after now these offset addr in our IDT data structure are relative to the wrapper_0 and now we need to turn them in absolute addr.

```nasm
; Relocating of IDT entries and setting IDTR
setup_idt:
 mov rax, wrapper_0

 ; bits 0..15 -> ax, 16..31 -> bx, 32..64 -> edx
 mov rbx, rax
 mov rdx, rax
 shr rdx, 32
 shr rbx, 16

 mov r10, idt ; pointer to the actual interrupt gate
 mov rcx, 255 ; counter
.loop:
 add [r10+0], ax
 adc [r10+6], bx
 adc [r10+8], edx
 add r10, 16
 dec rcx
 jge .loop

 lidt [idt_descr]
 ret
```

- At runtime we simply take the absolut addr of wrapper_0 and added it to each of these entries of the IDT
- lidt :  load the IDT register with this IDT decriptor(data structure here)
- now CPU knows  where in mem our IDT starts and how big is it.
- Wrpper function → how we sets up the intr decriptor table

### State Save

- Every CPU has internal state
    - represented as register contents
- x86-64 (we’re ignoring FPU + vector extensions here):
    - wrapper need to save all V-R

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%206.png)

- why not save RFLAGS(intr pointer)?
    - CPU automatically saves RFLAGS before any ctrl flow deviation
- **Total save (stack): 22 registers == 176 Bytes**

### Example (Linux arch/x86/entry/calling.h)

- Implemented as an assembly macro (used in many places)
    - analogously POP_REGS
    
    ![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%207.png)
    

### Example (Linux arch/x86/kernel/entry_64.S)

- Used e.g. in interrupt handlers:

```nasm
ENTRY(error_entry)
 UNWIND_HINT_FUNC
 cld
 PUSH_AND_CLEAR_REGS save_ret=1
 ENCODE_FRAME_POINTER 8
 /* ... */
 CALL_enter_from_user_mode
 ret
```

### Context Save: Who Does What ?

- Context save **in interrupt handlers**:
    - rflags, rsp, cs, ss and rip are automatically saved by the CPU
    - all other registers must be saved by the IRQ handler
        - either in the wrapper function (assembler)
        - or the compiler already generates code for this
- Context save **when calling functions**:
    - Solution 1: **Caller** saves all registers it still needs later(volatile)
    - Solution 2: **Callee** saves all registers it modifies(non-volatile)
    - Solution 3: One part of the registers is caller-saved, the other is callee-saved

### High-Level Language Context Save

- In practice, solution 3 is used
    - Generally, this depends on the compiler – but CPU manufacturers define a standard to guarantee interoperability on the binary level
- Partitioning into two subsets of registers
    - Volatile registers (caller-saved, scratch registers)
        - Compiler assumes the called function will modify contents
        - Caller must save (if it still needs contents)
        - x86-64: rax, rdi, rsi, rdx, rcx, r8–r11, FPU/SSE/AVX/
    - Non-volatile registers (callee-saved, non-scratch registers)
        - Compiler assumes the called function will not modify contents
        - Callee must save (if it does modify anyways)
        - x86-64: rbx, rsp, rbp, r12-15
    - Interrupt handlers must also save volatile registers!

### State Save in the Wrapper

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%208.png)

- macro:
    - rbp and rax are the first 2 volatile regs that we want to save.
    - mov the wrapper nr into al regsiter
        - guardian(): is the start point of IH activities. the guardian function has a parameter with the intr nr that was triggered.
        - for each of these possible 256 intrs that could comow in, there is 1 entry in the IDT, each of these entries points to a different handler function.
- wrapper_body
    - mov rdi, rax
        - rdi stores the 1. parameter in x86
    
    ![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%209.png)
    

### Repetition: References in C++

- Plugbox and Gate
- 

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%2010.png)

```cpp
// ASSIGN: Plug in a handler routine, provided in the form of a Gate obj.
void assign(unsigned int slot, Gate& gate);
// REPORT: Retrieve the Gate object for the specified slot.
Gate& report (unsigned int slot);
// if the reference are not here, this func will return to
 // a copy of a gate object
```

### How Do C++ References Work?

- Semantically: Aliases for objects
- Technically: Initialized, immutable pointers
    - Upon initialization of a reference, the compiler automatically takes the address of the initializing object.
    - When using a reference in an expression, automatically the referenced
    object is used.

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%2011.png)

# Lab Task #3

## Pro/Epilogue Model – Sequence Example

- guaranties that ctrl flow on one level is sequetilaized

![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%2012.png)

1. enter the epilogue level:  → corresponds to cli
2. Interrupt is signaled on level L1, execute prologue.
3. Prologue requests epilogue for delayed execution (relay).
4. Prologue terminates, interrupted L½ control flow (application) continues.
5. Application control flow leaves epilogue level L½ (leave), process meanwhile accumulated epilogues.
6. Epilogue terminates, application control flow continues on L0
- When do we have to process pending epilogues?
    - Just before the CPU returns to L0, While the application ctrl flow run on epilogue/processing epilogues/the CPU executed control flows on levels L1...n, epilogues may have accumulated.

## Pro/Epilogue Model

- what could happens when INTR been disabled all the time while the shared data structure is modifying?
    - might lose INTRs,the first intr can be stored into interrupts ctrl HW,  but the 2. and furthers intr may get lost.
    - INTR latency can get really high
        - if u cannot sevice the intr fast enough
    - Assenment:
        - acceptable when it’s been done quickly, probably
        - not okay when the condition is more complexer.
- what happens in Pro/Epilogue Model
    - Previously
    1. application run on L0
    2. once you get INTR, you end up in IH level L1
    3. INTRs are automatically been disabled when get into L1 (what cpu does)
    4. trigger() runs in L1
    5. return back to L0 and meanwhiles enable INTR
    - Pro/Epilogue Model
        
        ![Untitled](exercise-3%209158396fcee44175aecbd5897be0cb4b/Untitled%2012.png)
        
        1. appl want to do some modification with the buf[…] so it enter the epilogue level :  → enter() corresponds to cli
        2. Interrupt is signaled on level L1, execute prologue.
        3. Prologue requests epilogue for delayed execution (relay).
        4. Prologue terminates, interrupted L½ control flow (application) continues.
        5. Application control flow leaves epilogue level L½ (leave), process meanwhile accumulated epilogues.
        6. Epilogue terminates, application control flow continues on L0.

#
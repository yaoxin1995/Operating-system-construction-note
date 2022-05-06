# **week-4**

# **week-4**

- Course: OSC
- Materials: [https://os.inf.tu-dresden.de/Studium/OSC/SS2022/L05-IRQ-Sync.pdf](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/L04-IRQ-SW.pdf)

# **Lecture : Interrupts – Software**

## TERMs we may used for next lectures

- **3 categories**
    - **Instruction cannot be executed**
        - e .g PF, Protection fault, /0 ....
        - TRAP, EXCEPTION : Handling of an error or an exception condition
    - **Instruction triggers voluntary mode switch**
        - e.g syscall, breakpoint instructions
        - Supervisor Call, Software interrupt / trap : Call to an operating system function
    - **Device requires attention**
        - Timer alarm, Key pressed, “NMI”
        - Interrupt :  Reaction to an asynchronous external event

## why we need interrupts in first place?

- It would be a lot of useless work to have the kernel periodically poll the device in order to process it, because peripherals are generally slower to process than the CPU, and the CPU cannot wait for external events all the time. So being able to have the device actively notify the kernel when it needs it would be a smart way to go, and that is interrupts.
- another option is using busy-waitting:
    - + PRO
        - low latency
    - - CON
        - by busy waitting the cpu can’t do anything else.
        - only make sence when cpu hat only 1 task....

## Distinguish Trap and Interrupt

- Trap : triggerd by an instruction
    - trap or int instruction for syscalls
    - Undefined result(/0)
    - Hardware problem(bus error)
    - PF(need OS do sth)
    - Invaild instruction(programmier error)
    - Porperities
        - normally predictable, reproducible
        - Restart or abort the triggering activity
- INTR : triggerd by hardware
    - Hardware requires attention by software
    (Timer, Keyboard controller, Hard-disk controller, …)
    - Porperities
        - not predictable, not reproducible
        - resume the interrupted activity

## Basic Assumptions

- Assumption: Handler Routine : The CPU automatically starts the handler routine.
    - Necessitates dispatch to a handler : decide which handler is responsible for the intr request
    - classic solution is : Vector tables
        - Register contains vector-table start address: this table could start at the adress that defined in the cpu register
        - Instead adress the table can directly contain code, which usually jumps to the handler routine, cuz the space for actual machine code is normally limited.
        - In some architectures: programmable “event controller” handles interrupt in hardware
        - Table contains descriptors
        - Handler routine has own process context
- Assumption: Supervisor mode : Interrupt handling takes place in supervisor mode.
    - Only the OS may access devices without restrictions.
        - Before interrupt handling, the CPU switches to the privileged supervisor mode.
    - Interrupts are the only mechanism to preempt noncooperative applications.
    - but for some CPUs, the subdivision ( user and supervisor ) is not exist.
- Assumption: State Save : The interrupted program can be resumed.
    - OPTIONS
        - more information on cause/trigger on the stack
        - no priorities
        - special “interrupt stack”
        - State save in registers(make sence when nesting is not allowed)
- Assumption: Atomic Behavior : Machine instructions are atomic.
    - Defined CPU state when handler routine starts
    - State restorable
    - Ideally, all stages are always in use, i.e. multiple instructions are
    executed in parallel. When should we check whether an IRQ has
    been issued?
        - All instructions preceding the instruction indicated by the saved program counter have been executed and have modified the process state correctly
- Assumption: Interrupt Suppression : Interrupt handling can be disabled/suppressed.
    - WHY :Automatic suppression by the CPU before starting the handler routine?
        - 中断不能预测
        - a stack overflow would be possible
    - Hardware disable all INTR or INTR with lower and same priority.
    - option : Suppress interrupts that are currently being handled
        - cuz there will has low latency when without preference for indicidual devices.

## Saving State

- Both the main program and the interrupt service subroutine use the CPU's internal registers and other resources. In order for the interrupt handler not to destroy the contents of the registers in the main program, the contents of each register at the breakpoint should first be pressed into the stack to protect it
- Any state the interrupted program does not expect to asynchronously change
    - must not be modified in an interrupt handler or
    - must be saved, and restored afterwards
- total save
    - Handler routine saves all registers that were not automatically saved
        - con : probably saves too much
        - pro : + saved state easily accessible (one coherent data structure)
- Partial save
    - Handler only saves registers that
        - 1) are modified in the interrupt handler
        - 2) are not saved/restored by other parts of the handler*(*) or the functions it calls directly or indirectly
    - Transition to High-level language

![L401.png](https://github.com/yaoxin1995/OSC_NOTE/blob/week-4/image/L401.png)

- Volatile and Non-volatile Register
    - non-volatile Register: callee-saved register
        - Compiler guarantees that the stored value is conserved across function calls
        - callee is responsible for saving the register after using.
    - volatile Register: a caller-saved or scratch
        - caller must save/restore all by itself, is it still needs the value after function call.
- Restoring State
    - As its last duty, wrapper must restore saved register contents, and must not again modify them afterwards!
    - A special instruction (e.g. rte or iret) completes the restore procedure:
        - Reads automatically saved state from supervisor stack
        - Sets the saved CPU mode (user/supervisor), jumps to saved address

## Modifying State : main purpose of interrupt handling

- the problem should be noticed by modifying state:
    - not all simple instructions are atomic, it depends on CPU type, compiler, code optimizations.
    
    ![L402.png](https://github.com/yaoxin1995/OSC_NOTE/blob/week-4/image/L402.png)
    
    - **The memory of buffer is limited**, if input is not handled/consumed fast enough, the buffer can fill up. The interrupt handler routine cannot store further input there. This input is lost.
    - **Buffer data structure can “break”**
        - inconsistent intermediate states during modifications by regular control flow
        - write while reading
        - write on copy
    - regular control flow do not interrupt the INTR handler

## Synchronization Techniques

- “Hard” Synchronization
    - disable interrupt →avoid race condition
    
    ![L403.png](https://github.com/yaoxin1995/OSC_NOTE/blob/week-4/image/L403.png)
    
    - problem
        - Hazard of losing interrupt requests
        - High and difficult to predict “interrupt latency”
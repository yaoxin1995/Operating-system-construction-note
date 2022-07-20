# week-7

# **week-7**

- Course: OSC
- [Materials:](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/L07-Threads.pdf)

### Quasi Paralleliism

- Function calls cannot achieve quasi paralleism between 2 function exections
    - simple function calls always run to completion
    - recursive func calls also run to completion. so infinite recursion and stack overflow
        
        ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled.png)
        
- Funktions with the feature that can be left during execution and reentered again
    - programm counter is saved and restored with goto
    - but :
        - Direct jumps from and to functions is undefined in C!
        - State consists of more than the PC – what about registers, stack, …?
        

### Routine, Control Flow

- Routine is a finite sequence of instructions
    - function f
    - Language mechanism/abstraction in almost all programming languages
    - is executed by a (routine) ctrl flow
- (Routine) Ctrl flow:  a (routine) ctrl flow execution
    - excution and ctrl flow : synonyms
    - the execution <f> of func f
        - starts after activation with the first instruction of f
- <f>, <f'>, <f''> denote executions of function f.
- <f> call g
    - creates new execution <g> of g
    - suspends execution <f>
    - activates execution<g>(first instruction is executed)
- <g>ret
    - destroys excution <g>
    - reacticates excution of the creating/ calling ctrl flow
- Routines →Asymmetric Continuation Model
    
    ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%201.png)
    
    - Routine ctrl flow form a continuatly hierarchy
    - activated ctrl flow are continued following LIFO.
    - INTR
        - <f>: irq : like call, but implicit
        - <irq> iret: like ret
    
    ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%202.png)
    

### Coroutine

- Coroutine : generalized routine
    - additionally allows explicit suspend/resume
    - Supportes by serveral programmimng language
    - is executed by a coroutine ctrl flow
- Coroutine ctrl flow: a coroutine execution
    - control flow with own, independent state
        - stack, registers
        - in peinciple an independent thread
- create g
    - creates new coroutine execution<g> of g
- <f> resume <g>
    - suspends coroutine execution <f>
    - (re)acticates coroutine execution <g>
- destory <g>
    - destorys coroutine execution<g>
    
- Difference to routine control flows:
    
    Activation and re-activation are temporally decoupled from creation and destruction.
    
- Coroutines Symmetric Continuation Model
    - Coroutine ctrl flow form a continuation sequence
        
        ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%203.png)
        
        - Coroutine stats is conserved across sespensions/activations
        - all coroutine ctrl flows are equitable
            - Cooperative multitasking
            - Continuation order is arbitrary
- in this course we understand coroutine as a technical to
    - implement multi-threaading in the OS
    

## Implementing Coroutines

- Continuations
    - Continuation is actually  an object  → Rest / remainder of an execution
        - an object that represents a suspended control flow
            - Program counter, registers. local variables….
            - complete control-flow state
        - needed to reactivate the control flow
        

- Routines → Asymmetric Continuation Model
    - NV-R will not be modified across the function calls, if function wanna modify those registers, function must save it on the stack
    - Routine continuation are instantiated on the stack
        - in the form of stack frames, created and destoryed by
            - compiler(and CPU) with call, ret
            - wrapper function(and CPU) at interrupt, iret
        - Stack is provided by the HW (CPU stack)
            - Instructions like call, ret, push, pop implicitly use this stack
            
            ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%204.png)
            
        1. balck part →f
        2. f calling g, calling struc sace PC and jumps to g and red part(red is belonging to g stack frame)
        3. g calling h
        - Callee hold the the continuation of calling function
- Coroutines →Symmetric Continuation Model(diffrently)
    - A coroutine ctrl flow needs an own stack
        - for local variables:  they are part of its state
        - for sub routine calls :
        - During execution, this stack is the CPU stack.
        
        coroutine ctrl flows can create routine ctrl flow on their stack and activate them
        
    - **APPROACH:** coroutine continuations are instantiated as stack frames on their stack.
        - A ctrl flow context is represented by the stack
        - The top stack element always contains the continuation
        - A ctrl flow switch corresponds to a stack swich and ‘return’
        
        in principle, this approach implements coroutine continuations using routine continuations
        
- **resume**
    - Task: Switch the coroutine control flow
        - caller of resume : pushes pc on the stack → in resume() might push additonal value on the stack→ switch sp to another stack.
        - when g-coroutine leave off , call resume, the top most value of the g stack is the PC of g-coroutine, afterward return: pops the value and contine g-coroutine
        
        ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%205.png)
        
    
    ```cpp
    // Stack-pointer type (the stack is an array of void*)
    typedef void** SP;
    extern "C" void resume( SP& from_sp, SP& to_sp ) {
    /* current stack frame is the continuation of the
    to-be-suspended control flow (caller of resume) */
    < save CPU stack pointer in from_sp >
    < load CPU stack pointer from to_sp >
    /* current stack frame is the continuation of the
    to-be-(re)activated control flow */
    } // return
    ```
    
    - Problem: The stack frame does not contain any non-volatile registers→ so they must be explicitly saved and restored
    - solution
    
    ```cpp
    extern "C" void resume( SP& from_sp, SP& to_sp ) {
     /* current stack frame is the continuation of the
     to-be-suspended control flow (caller of resume) */
     < push non-volatile registers on the stack >
     < save CPU stack pointer in from_sp >
     < load CPU stack pointer from to_sp >
     < pop non-volatile registers from the stack >
     /* current stack frame is the continuation of the
     to-be-(re)activated control flow */
    } // return
    ```
    
    - architecture of resume
        - Stackframe structure
        - NV-R
        - stack growth direction
    
    ```cpp
    Example Motorola 68000
    // extern "C" void resume( SP& sp_from, SP& sp_to )
    resume:
     move.l 4(sp), a0 // a0 = &sp_from
     move.l 8(sp), a1 // a1 = &sp_to
     movem.l d2-d7/a2-a6, -(sp) // nv registers to stack
     move.l sp, (a0) // sp_from = sp
     move.l (a1), sp // sp = sp_to
     movem.l (sp)+, d2-d7/a2-a6 // load nv regs. from stack
     rts // “return
    ```
    
    ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%206.png)
    
- **create:** Create coroutine control flow <start>
    - object we need:
        - Stack mem :
        
        ```cpp
        static void *stack_start[ 256 ];
        ```
        
        - a stack pointer
        
        ```cpp
        SP sp_start = &stack_start[ 256 ];
        ```
        
        - a start function
        
        ```cpp
        void start( void* param ){...}
        ```
        
        - parameters for the start function

  

- **Approach: create generates two stack frames**
    - the start function’s frame
    - resume’s frame(contains start function’s continuation)

```cpp
Example Motorola 68000
void create( SP& sp_new, void (*start)(void*), void* param) {
	 *(--sp_new) = param; // start-function parameter
	 *(--sp_new) = 0; // (non-existent) caller’s return addr.
	 *(--sp_new) = start; // start() address
	 sp_new -= 11; // n-v. registers (values don’t matter)
}
```

![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%207.png)

- **destroy :destroy coroutine control flow**
    - deallocate control-flow context
        - free the context variable (stack pointer)
        - Stack memory can be used otherwise afterwards.
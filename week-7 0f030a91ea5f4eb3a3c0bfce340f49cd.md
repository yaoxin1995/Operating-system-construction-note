# week-7

# **week-7**

- Course: OSC
- [Materials:](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/L07-Threads.pdf)

### Quasi Paralleliism

- Function calls cannot achieve quasi paralleism between 2 function executions
    - simple function calls always run to completion
    - recursive func calls also run to completion. so infinite recursion and stack overflow
        
        ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled.png)
        
- Funktions with the feature that can be left during execution and reentered again
    - program counter is saved and restored with goto
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
        - the most recently activated ctrl flow always terminates first.
        - caller is onlz resumed after callee terminates
    - INTR
        - <f>: irq : like call, but implicitly
        - <irq> iret: like ret
        - Intr can be understood as implicitly created and acticted routine excutions
    
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
    
    Activation and re-activation are temporally decoupled from creation and destruction.(|什么意思???)
    
- Coroutines Symmetric Continuation Model
    - Coroutine ctrl flow form a continuation sequence
        
        ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%203.png)
        
        - Coroutine stats is conserved across sespensions/activations
    - all coroutine ctrl flows are equitable
        - Cooperative multitasking
        - Continuation order is arbitrary
- in this course we understand coroutine as a technical to
    - implement multi-threading in the OS
    

## Implementing Coroutines

- Continuations
    - Continuation is actually  an object  → Rest / remainder of an execution of a routine
        - an object that represents a suspended control flow, it contains …
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
            
        1. balck part →routine excution <f>
        2. f calling g, calling structure saves PC and jumps to g and red part(red is belonging to g stack frame)
        3. g calling h
        - Callee hold the the continuation of calling function
        - essential: the return adr and para. logically belong to the excution stck frame but it’ s created by its caller. and the funktion itself create is start form where its frame pointer point to.
- Coroutines →Symmetric Continuation Model(diffrently)
    - Each coroutine ctrl flow needs an own stack
        - for local variables:  they are part of its state
        - for sub routine calls : form coroutine we still want to call regular routines from the same stack. routine is sort of like a subordinate coroutine , coroutine with the coroutine stack is kind of like a container for asymmetric func hierarchie. so in top level you have a coroutine but coroutine may call routines.
        - During execution of coroutine, this stack is the CPU stack. once u switch to diffrent stack, the stack pointer  switches one stack to another.
        - Thus, coroutine ctrl flows can create routine ctrl flow on their stack and activate them
    - **APPROACH:** coroutine continuations are instantiated as stack frames on their stack.
        - A ctrl flow context is represented by the stack
        - The top stack element always contains the continuation
        - A ctrl flow switch corresponds to a stack swich and ‘return’
        
        in principle, this approach implements coroutine continuations using routine continuations
        
- **Implementation : resume**
    - Task: Switch the coroutine control flow
        - caller of resume : pushes pc on the stack → in resume() might push additonal value on the stack→ switch sp to another stack.
        - when g-coroutine leave off , call resume, the top most value of the g stack is the PC of g-coroutine, afterward return: pops the value and contine g-coroutine
        
        ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%205.png)
        
    
    ```cpp
    // Stack-pointer type (the stack is an array of void*)
    typedef void** SP;
    extern "C" void resume( SP& from_sp, SP& to_sp ) {
    /* current stack frame is the continuation of the
    to-be-suspended control flow (caller of resume): f coroutine*/
    < save CPU stack pointer in from_sp >
    < load CPU stack pointer from to_sp >
    /* current stack frame is the continuation of the
    to-be-(re)activated control flow : g coroutine*/
    } // return
    ```
    
    - **Problem**: The stack frame does not contain any non-volatile registers→ so they must be explicitly saved and restored, otherwise g function may modify the NV-R.
    - **solution :**
        - save NV-R regs to a special data structure(oostubs)
        - save them as local variables on the stack.(code below)
    
    ```cpp
    extern "C" void resume( SP& from_sp, SP& to_sp ) {
     /* current stack frame is the continuation of the
     to-be-suspended control flow (caller of resume) */
     < push non-volatile registers on the stack >
     < save CPU stack pointer in from_sp >  // sp_from = sp 
     < load CPU stack pointer from to_sp >  // sp = sp_to
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
    
    ![0808DAE5-D403-4C80-ABE9-55314D26B4FC.jpeg](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/0808DAE5-D403-4C80-ABE9-55314D26B4FC.jpeg)
    
    ![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%206.png)
    
- **Implement: create:** Create coroutine control flow <start>
    - object we need:
        - Stack memory :
        
        ```cpp
        static void *stack_start[ 256 ];
        ```
        
        - a stack pointer
        
        ```cpp
        SP sp_start = &stack_start[ 256 ];
        ```
        
        - a start function (kickoff in oostubs)
        
        ```cpp
        void start( void* param ){...}
        ```
        
        - parameters for the start function: in oostubs is a pointer to a coroutine object we want to excuted.

  

- **Approach: create generates two stack frames**
    - as if thr start function has already call resume before
        - the start function’s frame
        - resume’s frame(contains start function’s continuation)
            - NV-R
            - PC
            - sp_from, sp_to
    - create coroutine stack , it contains
        - initial value
        - SP
        - potential data structure with all NV-R
        - make stack more or less looks like the function already called resume before

```cpp
Example Motorola 68000(in oostubs is toc_settle) 
void create( SP& sp_new, void (*start)(void*), void* param) {
	 *(--sp_new) = param; // start-function parameter
	 *(--sp_new) = 0; // (non-existent) caller’s return addr.
	 *(--sp_new) = start; // start() address
	 sp_new -= 11; // n-v. registers (values don’t matter)
}
```

![Untitled](week-7%200f030a91ea5f4eb3a3c0bfce340f49cd/Untitled%207.png)

- parameter could be e.g the pointer to the coroutine, and within the start of the coroutine you can call some action method in appl as a routine.

- **destroy :destroy coroutine control flow**
    - Approach : deallocate control-flow context
        - free the context variable (stack pointer)
        - Stack memory can be used otherwise afterwards.

## Coroutines as a Basis for Multithreading

### Kernel-Level Threads

- Prerequisite for multitasking is, Cooperation
    - Applications must be implemented as coroutines
    - Applications must know each other
        - build some kind of ring for coroutine one passed to next one
    - Applications must activate each other
- Alternative: Perceive “cooperation capability” as an operating-system responsibility
- **Approach: Run applications “unnoticeably” as independent threads**
    - OS takes care of creating coroutine control flows
        - Each application is called as a routine from an OS coroutine
        - consequently, indirectly every application is implemented as a coroutine
    - OS takes care of suspending running coroutine control flows
        - so that applications do not have to be cooperative
        - necessitates a preemption mechanism
    - OS takes care of selecting the next coroutine control flow
        - so that applications do not have to know each other
        - necessitates a scheduler
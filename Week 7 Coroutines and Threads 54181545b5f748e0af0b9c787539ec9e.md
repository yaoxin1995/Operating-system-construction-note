# Week 7 Coroutines and Threads

Created: July 14, 2022 8:29 AM
Reviewed: No

## ****Quasi Paralleliism****

1. What is ****Quasi Paralleliism?****
    
    use function calls and other user level primitive (without OS involvement) to emulate the thread ****Paralleliism****
    
2. Why ****Quasi Paralleliism doesn’t work?****
    - simple function calls (experiments 1 and 2) always run to completion
    - recursive function calls (experiment 3) have the same problem, and thus cause infinite recursion and stack overflow
    - experiment 4 seems to work, but
        - C language doesn’t support such primitive, e.g., Direct jumps from and to functions,
        - and for context switching the state may consist of more than the PC (register, stack)
        
        ![Untitled](Week%207%20Coroutines%20and%20Threads%2054181545b5f748e0af0b9c787539ec9e/Untitled.png)
        
    

## Basic Terminology

### Routine and control flow

1. What is Routine and control flow
    - Routine is a finite sequence of instructions, i.e., Functions, and it is executed by routine control flow
    - (Routine) Control flow is a (routine) execution. e.g. the execution <f> of function f
    - Routines and executions have a schema-instance relationship and we use following symbols to distinguish routine and routine control flow
        
        `<f>, <f'>, <f''>` denote executions of function `f.`
        
2. Basic Terminology ():
    
    Routine control flows are created, managed and destroyed
    with specific primitives:
    
    - <f> call g (Execution <f> reaches instruction call g)
        - creates new execution <g> of g
        - suspends execution <f>
        - activates execution <g> (first instruction is executed)
    - <g> ret (Execution <g> reaches instruction ret)
        - destroys execution <g>
        - reactivates execution of the creating/calling control flow
3. Routines is a **Asymmetric Continuation Model**
    
    ![Untitled](Week%207%20Coroutines%20and%20Threads%2054181545b5f748e0af0b9c787539ec9e/Untitled%201.png)
    
    - Routine control flows form a continuation hierarchy
    - Activated control flows are continued following LIFO , the execution order of each activated control flow is LIFO. like a stack
    
    ![Untitled](Week%207%20Coroutines%20and%20Threads%2054181545b5f748e0af0b9c787539ec9e/Untitled%202.png)
    

### Coroutine, Control Flow and Thread

1. What is Coroutine and  Coroutine control flow
    - Coroutine  is generalized routine, it is executed by a coroutine control flow, and it  **additionally** allows explicit suspend/resume
    - Coroutine control flow is a coroutine execution, and each Control flow has its own, independent state (register, stack …)
    - Coroutines and coroutine control flows also have a schema instance relationship.
2. Basic Terminology:
    
    Coroutine control flows are created, managed and destroyed by additional primitives:
    
    - create g
        - **creates**                   new coroutine execution <g> of g
    - resume
        - **suspends**                coroutine execution
        - (re)**acticates**            coroutine execution
    - destory
        - **destorys**                  coroutine execution
3. Coroutines is **Symmetric Continuation Model**
    
    ![Untitled](Week%207%20Coroutines%20and%20Threads%2054181545b5f748e0af0b9c787539ec9e/Untitled%203.png)
    

- Each coroutine ctrl flow form a continuation sequence.
    - Coroutine state is conserved across suspensions/activations
- all coroutine ctrl flows are equitable (公平的)
    - Cooperative multitasking
    - Continuation order is arbitrary
- Coroutine control flows are often also called cooperative threads

**Q： Difference between coroutines and routine?**

- Coroutine  is generalized routine, it  **additionally** allows explicit suspend/resume
- They have different control flow semantic. Compare to routine control flows, in the coroutine control flows, activation is temporally decoupled from creation,  and re-activation is temporally decoupled from destruction
- Coroutines is **Symmetric Continuation Model,** Routines is a **Asymmetric Continuation Model**
- Routine control flows form a continuation hierarchy, but each coroutine ctrl flow form a continuation sequence.

## Implementing Coroutines

1. What is **Continuation?**
    
    Continuation represents rest / remainder of an execution. In other word, it is an object (control-flow state) that represents a suspended control flow. It is needed to reactivate the control flow
    
2. What is Routine continuations (the on stack stored state  is used for function calls)?
    
    Routine continuations are instantiated on the stack in the form of stack frame. The stack frame is created and destroyed by compiler (and CPU) with call, ret, or wrapper function (and CPU) at interrupt, `iret`
    
3. What is **Coroutine continuations** (the on stack stored state is used for thread switch)?
    
    ![Untitled](Week%207%20Coroutines%20and%20Threads%2054181545b5f748e0af0b9c787539ec9e/Untitled%204.png)
    
    - Each  coroutine control flow has its own stack. And the coroutine continuations are instantiated as stack frames on their stack.
        - Thus coroutine control flows can create routine control flows on their stack, and activate them!
    - The top-most stack element always contains the continuation.
    - A coroutine  control-flow context is represented by the stack.
    - A coroutine  control-flow switch corresponds to a stack switch and “return”.
    
4. resume() : Switch the coroutine control flow
    
    ```cpp
    // Stack-pointer type (the stack is an array of void*)
    typedef void** SP;
    extern "C" void resume( SP& from_sp, SP& to_sp ) {
    		/* current stack frame is the continuation of the to-be-suspended control flow (caller of resume) */
    	< push non-volatile registers on the stack >
    	< save CPU stack pointer in from_sp >
    	< load CPU stack pointer from to_sp >
    	< pop non-volatile registers from the stack >
    	/* current stack frame is the continuation of the to-be-(re)activated control flow */
    } // return
    ```
    
    **Q: Why dose resume have to save non-volatile registers to stack before context switch?**
    
    Assume we have function f (belong to coroutine control flow A), and we want to switch from coroutine control flow A to B. In f we do the context switch by calling resume. In this case, f is caller and resume() is callee. According to the c calling convention, the caller expects non-volatile register not to be modified cross the function calls. If we don’t save those registers before context switch, coroutine B may modify those registers and cause the function f can’t be resume correctly.
    
    ![Untitled](Week%207%20Coroutines%20and%20Threads%2054181545b5f748e0af0b9c787539ec9e/Untitled%205.png)
    
5. create(): Create coroutine control flow <start>
    - We need:
        - Stack mem : `static void *stack_start[ 256 ];`
        - a stack pointer`SP sp_start = &stack_start[ 256 ];`
        - a start function `void start( void* param ){...}`
        - parameters for the start function
    - We create the coroutine control flow in suspended state
        - Stack represents the context
        - Execution should not start until resume is called
    - **Approach: create generates two stack frames**
        - the start function’s frame
        - resume’s frame(contains start function’s continuation)
    
    ```cpp
    Example Motorola 68000
    void create( SP& sp_new, void (*start)(void*), void* param) {
    	 *(--sp_new) = param; // start-function parameter
    	 *(--sp_new) = 0;    // (non-existent) caller’s return addr.
    	 *(--sp_new) = start; // start() address
    	 sp_new -= 11;       // n-v. registers (values don’t matter)
    }
    ```
    
    ![Untitled](Week%207%20Coroutines%20and%20Threads%2054181545b5f748e0af0b9c787539ec9e/Untitled%206.png)
    
    1. Resume load NV-R
    2. Resume use ret (pop eip) to return to the beginning of start function
    3. Represents the return address of the start function. Since the start function is the very first function in the coroutine control flow, the return address should be 0. 
6. destroy: destroy coroutine control flow by deallocating the control-flow context (stack)

 

![Untitled](Week%207%20Coroutines%20and%20Threads%2054181545b5f748e0af0b9c787539ec9e/Untitled%207.png)
# week 5 Interrupts – Synchronization

Created: July 19, 2022 10:51 PM
Reviewed: No

# Lecture 5

### Why ensuring consistency between an application control flow (A) and
an interrupt handler (IH) works differently than between processes?

- Relationship between A and IH is asymmetric
    - an application control flow and interrupt handling are different kinds” of control flows, i.e.,
    - IH interrupts A
        - implicitly, at an arbitrary point
        - always higher priority, runs to completion
    - A can suppress IH (better: delay)
        - explicitly, with `cli / sti`
- Synchronization / maintenance of consistency is one-sided

### Control-Flow Level Model

![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled.png)

- Arbitrary sequentialization strategy possible (FIFO, LIFO, with priorities, random), and the strategy for level 2-n is defined by Interrupt controller (PIC)
- Each state variable is (logically) assigned to exactly one level Lf
    - Accesses from Lf implicitly consistent ( sequentialization)
    - For accesses from higher/lower levels, we must explicitly maintain consistency
- Measures for maintaining consistency:
    - **“from above” (from Le with e < f) with hard synchronization**, for example, thread in L0 want to access data structure on L1
        - explicitly switch to Lf for the access (delay)
        - Thereby, the access comes from the same level. ( sequentialization)
    - **“from below” (from Lg with f < g) with nonblocking synchronization**,  for example, thread in L1 want to access data structure on L0
        - make sure algorithmically that interrupts do not endanger consistency
        - the nonblocking synchronization need interrupt-transparent algorithms
        

### Hard Synchronization (access data structure on level f from level e with e < f)

![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%201.png)

- Definition: access data structure on level f from level e with e < f using `cli` and `sti`
- Pro:
    - Maintains consistency : for complex data structures and access patterns also works and independent for compiler.
    - Simple to use (for the developer), always works
- Con:
    - Broadband effect : delay all interrupt handlers ctrl flows on same and lower level(bigger Nr)
    - Priority violation : delay ctrl flows with high priority.
- Assessment:
    - Whether disadvantages become significant depends on the delays’
        - frequency
        - mean duration, and
        - maximum duration
    - Maximum duration is the most critical one, because
        - directly influences the (to be expected) latency
        - Latency too high can cause data can get lost
            - Interrupts aren’t noticed
            - Data is picked up too slowly from I/O devices
        
- Conclusion: Hard synchronization is rather unsuitable for maintaining consistency of complex data structures

### Nonblocking Synchronization

- Definition: access data structure on level e from level f with e < f using variant algorithm
    
    ![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%202.png)
    
- Consistency condition:
    - The result of an interrupted execution must be equivalent to an arbitrary sequential execution of the operations, for example: in case of consumer( on level L0) and producer (interrupt handler in L1) scenarios:
        - either consume() before produce() or consume() after produce()
- **See ppt to view concrete example**
- Advantages:
    - Maintains consistency (by interrupt transparency)
    - No priority violations (interrupts stay enabled!)
    - No cost, or only in the (rare) conflict situation
        - no cost →bounded-buffer example
        - in the conflict situation→optimistic approaches, system-time example (additional cost by restarting)
- Disadvantage:
    - Complexity (need to a special algorithm for each concrete scenarios)
        - If we find an algorithm at all, it’s usually hard to understand and even harder to verify
    - Constraints
        - Tiny code changes can ruin the consistency guarantee
        - Compiler’s code generation must be taken into account (compiler may reorder the code due to optimization)
    - Predictability
        - Costs for restart unpredictable for large amounts of data (有可能需要轮询很多的data)
        
- Conclusion:
    - Nonblocking synchronization is neat. However, the involved algorithms are special solutions for special cases.
    - It is not suitable as a generally applicable measure for maintaining consistency of complex data structures

### Synchronization with the Prologue/Epilogue Model

- Motivation
    
    Hard synchronisation is simple and always works, but it cause 
    
    - long delay when accessing state from the level with smaller Nr, because we delay other app running on level L0
    - long interrupt delay when long modifying state in IH itself, because IH itself delay other interrupt on same level
    
    The reason for the delay is that the shared data structure resides on level 1(hardware interrupt level)
    
    ![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%203.png)
    
- Solution:  insert another level L½ between application level L0 and the interrupt-handling levels L1...n
    
    ![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%204.png)
    
    - add another level named L½→ Epilogue Level
        
        ![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%205.png)
        
    - IH is divided into prologue and epilogue
        - interrupt handling start in their prologue (always) and are continued in their epilogue (on demand)
        - Prologue
            - runs on interrupt level L1...n
            - is short, touches little or no state
                - Hardware interrupts are only disabled briefly
            - can request an epilogue on demand
        - Epilogue
            - runs on (software) level L½ and the execution is delayed in respect to prologue
            - does the actual work
            - has access to most of the state, i.e., State resides (as far as possible) on here
- Implementation
    - Basic operation:
        - app explicitly enter the epilogue level: enter()
            - corresponds to cli in hard synchronization
        - app explicitly leave the epilogue level and go back to L0: `leave()`
            - corresponds to sti in hard synchronization
        - request an epilogue: `relay()`
            - corresponds to pulling an IRQ line to “high” at the PIC (high in PIC mean there is a interrupt req need to be handled)
    - Basic work flow
        
        ![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%206.png)
        
        1. App enter the epilogue level: → corresponds to cli
        2. Interrupt is signaled on level L1, execute prologue.
        3. Prologue requests epilogue for delayed execution (relay).
        4. Prologue terminates, interrupted L½ control flow (application) continues.
        5. Application control flow leaves epilogue level L½ (leave), process meanwhile accumulated epilogues.
        6. Epilogue terminates, application control flow continues on L0
    - When do we have to process pending epilogues?
        
        In short, just before the CPU returns to L0. But we need to consider 3 concrete scenarios where we need to handle epilogues:
        
        1. when explicitly leaving the epilogue level with leave()
            - While the application control flow ran on epilogue level, more epilogues
            may have accumulated ( sequentialization)
        2. after processing the last epilogue
            - While processing epilogues, more epilogues may have accumulated.
        3. after the last interrupt handler terminates
            - If we have a multi. level interrupt system, interrupt on higher level can interrupts on lower level. Therefore, epilogues may have accumulated. After the last interrupt handler terminates (CPU control flow will go back from level 1…n to 0), we have to execute padding epilogues,
    - Two implementation variants :
        - with hardware support via an AST
            
            ![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%207.png)
            
            ```cpp
            //L1 is now where AST been implemented
            //L2 is where HW intr start
            void enter() {
             CPU::setIRQL(1); // setting level to 1, delay AST
            }
            void leave() {
             CPU::setIRQL(0); // allow AST (pending
            } // AST would now be processed)
            void relay(<Epilogue>) {
             <enqueue epilogue in queue>
             CPU_SRC1::trigger(); // activate level-1 IRQ (AST)
            }
            void __attribute__((interrupt_handler)) irq1Handler() {
             while (<Epilogue in queue>) { 
             <dequeue epilogue from queue>
             <process epilogue>
            } }
            ```
            
        - completely software-based
    - Where are the padding epilogues stored?
        
        ![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%208.png)
        
        - The the padding epilogues are stored in the epilogue queue that is resides on L1
        - Again the queue is shared between L1/2 and L1.  In other words, when interrupt happens and trigger prologue,  which calls relay() to launch the epilogue or store the epilogue in the queue.  On the other hand,  after CPU executed epilogue and want to go back to level 0 (leave()), it need to check whether we still have padding epilogues in the queue.  Therefore we have to synchronize the access to the queue.
            - Hard synchronisation:  put queue on L1, and do hard synchronisation when access this queue from L1/2.
            - nonblocking synchronization: put queue on L1/2 and do nonblocking synchronization when access the queue from level 1
                - enqueue can interrupt dequeue
            - Assessment: Hard synchronization seems acceptable here, since the time frame with interrupts disabled (runtime of dequeue()) is short and deterministic.
- Assessment
    - Advantages
        - Maintains consistency (synchronization on epilogue level)
        - Programming model corresponds to the (easily understandable)model behind hard synchronization
        - Also complex state can be synchronized
            - without losing any IRQs
            - protect all kernel states
        - 
    - Disadvantages
        - Additional level→ additional overhead
            
            ● Epilogue activation could take longer than direct handling
            ● Higher complexity for the OS developer
            
        - We don’t completely get rid of disabling interrupts
            - Shared state between prologue and epilogue must still be synchronized hard or nonblockingly (the epilogue queue)
    - Conclusion:
        - a good compromise for synchronizing accesses to kernel state.
        - suitable for maintaining consistency of complex data structures.

![Untitled](week%205%20Interrupts%20%E2%80%93%20Synchronization%20aa85facea6ad444191a5d183cc41ff00/Untitled%209.png)
# Week 9 Thread Synchronization

Created: August 6, 2022 2:23 PM
Reviewed: No

## Control-flow Level Model with Threads

1. Difference between Interrupt synchronization and thread synchronization
    
     Interrupt synchronization is the Synchronization of accesses by control flows from **different levels**
    
    - State was logically assigned to one specific level
    - Synchronization either “from above” (hard) or “from below” (non-blocking)
    - Implicit sequentialization within the same level
    
     Thread synchronization is Synchronization of accesses by control flows from the **same level**
    
2. New Control-Flow Level Model
    
    ![Untitled](Week%209%20Thread%20Synchronization%2006e8b603d2e045148cbded290955e549/Untitled.png)
    
3. Difference between old and new control flow model
    
    The new model support preemption on level 0. By supporting preemptive threads we cannot sustain the second assumption any longer (never interrupted by control flows on Le)
    
    - State accesses (from the level 0) are not implicitly sequentialized anymore
    - No run-to-completion semantics on level 0 anymore

## Thread Synchronization

1. Thread Synchronization assumption****:**** Thread can be preemptes by any thread at any time
2. Thread Synchronization ********Goal : Coordination of resource accesses

### Mutex – Mutual Exclusion

1. Mutex in general
    - It is an algorithm for enforcing mutual exclusion in a critical section
    - Mutex is a 2 side synchronization mechanism
        
        ![Untitled](Week%209%20Thread%20Synchronization%2006e8b603d2e045148cbded290955e549/Untitled%201.png)
        
    - Correctness hold if At every point in time, there is at maximum one thread in the critical section:
        
        ![Untitled](Week%209%20Thread%20Synchronization%2006e8b603d2e045148cbded290955e549/Untitled%202.png)
        
2. Mutex variant 1: Mutex with Busy Waiting
    - Implemented purely at user level, and wait busily in lock() until variable `locked` is 0
        
        ```cpp
        // __atomic_test_and_set is a gcc builtin for
        // (CPU specific) test-and-set
        class SpinningMutex {
        char locked;
        public:
        SpinningMutex() : locked (0) {}
        
        void lock(){
        	 while (__atomic_test_and_set(&locked, __ATOMIC_RELAXED));
        }
        
        void unlock() {
        	 locked = 0;
        }
        };
        ```
        
    - Assessment
        - Pro
            - Maintains consistency, satisfies correctness condition
            - Synchronization without involving the OS
            - Easy to implement
        - Con
            - Busy waiting wastes a lot of CPU time, especially for single CPU case where the busy waiting prevent scheduler from schedule other threads in the ready list
            - Only make sence on multiprocessor case
3. Mutex variant 1: Mutex with “Hard Synchronization”
    - General idea:
        - Deactivate multitasking before entering the critical section
        - Reactivate multitasking after leaving the critical section
    - Variant implementations:
        - Modify the scheduler
            - create a special, non-preemptible “real-time priority”, which is not preemptible ,etc.
        - or simply on epilogue level
            - Epilogue-level control flows are sequentialized
            - As long as a thread is on epilogue level, it cannot be preempted
    - Assessment
        - Pro
            - Maintains consistency, satisfies correctness condition
            - Simple to implement
        - Con
            - Broadband effect
                - Across-the-board delay of all threads (and potentially even epilogues!)
            - Priority violation
                - delay control flows with higher priority
            - **Pessimistic ?????**
                - We put up with the disadvantages, although the collision probability is very low.
        - Conclusions
            
            Thread synchronization on epilogue level has many disadvantages. It is, however, appropriate for very short, seldomly entered critical sections
            

### Passive Waiting

1. Q: **Why not mutex with busy waiting and hard synchronization**
    - Mutex with busy waiting: wastes CPU time
    - Mutex with hard synchronization: coarse-grained, violating priorities
    - Better approach: Exclude thread from CPU scheduling as long as the mutex is locked, and wake it up when the lock is free (**passive waiting**)
2. **What is passive waiting?**
    - Threads can “wait passively” for an event
        - Wait passively → be excluded from CPU scheduling
        - New thread state: **waiting (for an event)**
    - Occurrence of an event triggers leaving the waiting state
        - Thread is included in CPU scheduling
        - Thread state: **ready**
3. How to implements **passive waiting?**
    - 2 additional operation for Scheduler
        - **block(Waitingroom& w)**
            - enqueue active thread (caller) in queue of synchronization object w
            - activate another thread (from ready list)
        - **wakeup(Customer& t)**
            - enqueue t in ready list
    - Synchronization object:  **Waitingroom representing the event to wait for, usually it has a waiting queue of waiting threads**
        - enqueue(Customer*)
        - Customer* dequeue()
        
        ![Untitled](Week%209%20Thread%20Synchronization%2006e8b603d2e045148cbded290955e549/Untitled%203.png)
        
    - Implementation of Mutex using waiting group, which achieve passive waiting
        
        ```cpp
        class WaitingMutex : public Waitingroom {
         char locked;
        public:
         WaitingMutex() : locked(0) {}
         void lock() {
         while (__atomic_test_and_set(&locked, __ATOMIC_RELAXED))
         scheduler.block(*this);
         }
         void unlock() {
         locked = 0;
         // fetch possibly waiting thread from the queue ownd by waiting group 
        // and wake it up
         Customer *t = dequeue();
         if (t)
         scheduler.wakeup(*t); 
        //still test if lock free , if not, block again
         }
        };
        ```
        
        - Problem: Lock after free since the lock and unlock are the critical sections themselves
            
            Assume we have thread 1 and thread 2. Thread 1 is in the critical section and sleep, while thread 2 try to get the lock. The thread 2 execute the lock() and enter the while loop body since the variable lock is not 0. At this point, the scheduler decide to put it into sleep and wake up thread 1. Thread 1 then leave the critical section by calling unlock(). Later, the scheduler wake up thread 2, and thread 2 would be blocked although the lock is free.
            
        - Solution
            - Can’t use mutex using passive waiting → recursive block
            - Can use spin mutex/ hard synchronization based mutex to protect lock and unlock
        - Use hard synchronization based mutex to protect lock and unlock
            
            ![Untitled](Week%209%20Thread%20Synchronization%2006e8b603d2e045148cbded290955e549/Untitled%204.png)
            
        - Conclusion
            - Mutex state resides in the kernel on epilogue level as the scheduler state is on the epilogue level
            - Implementation of synchronization mechanisms for L0 control flows is synchronized on L½
        
        ![Untitled](Week%209%20Thread%20Synchronization%2006e8b603d2e045148cbded290955e549/Untitled%205.png)
        

![Untitled](Week%209%20Thread%20Synchronization%2006e8b603d2e045148cbded290955e549/Untitled%206.png)

After T0 got the lock and in the critical section, timer interrupt happens and calls resume to switch to T1, T1 goes to sleep since it can’t get the lock. Then T0 will be waked up and leave the critical section

### ****Semaphore****

1. What is semaphore?
    
    synchronization object + counter, if counter is lower that 0, then caller would be put into the waiting group and sleep. Later if the counter is raised, then one thread in the waiting group will be waked up (FIFO)
    
2. Semaphore primitive
    - P(), wait(), down()
        - if counter > 0, decrease counter
        - if counter ≤ 0, wait until counter > 0 and retry
    - V(), signal(), up()
        - increase counter
        - if counter = 1, wake up possibly waiting thread
3. Basic usage: producer/consumer scenarios
    - Producer calls V() for each produced element.
    - Consumer calls P() to consume an element, possibly waits.
4. **Q: Semaphore vs. Mutex**
    - Mutex is often understood as a two-valued semaphore
    - However, the semantics are different
        - A locked mutex (implicitly or explicitly) has an owner
            - Only this owner may call unlock()
            - Mutex implementations e.g. on Linux or Windows check this.
        - A mutex can (usually) also be locked recursively.
            - Internal counter: The same thread may call lock() multiple times; after a matching number of unlock() calls, the mutex is unlocked again.
    - In contrast, a semaphore can be incremented or decremented by any thread

## Summary

![Untitled](Week%209%20Thread%20Synchronization%2006e8b603d2e045148cbded290955e549/Untitled%207.png)
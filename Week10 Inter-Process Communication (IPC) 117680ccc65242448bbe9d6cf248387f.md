# Week10 Inter-Process Communication (IPC)

Created: August 7, 2022 2:57 PM
Reviewed: No

## ****Communication and Synchronization****

1. Q: **What is the relationship between communication and synchronization?**
    
    communication and synchronization are are related through the principle of **causality (**If A needs a piece of information from B to continue its work, A must wait until B supplies that information), which means **Message-based communication (usually) implies synchronization between sender and receive**r. The conmen synchronization primitives are a suitable basis for
    implementing communication primitives （semaphore）
    

## IPC via Shared Memory

### Positive properties

- **Atomic memory accesses do not require additional synchronization**
- Fast: **zero-copy**
    - put a message in shared mem, it doesn’t have to copy again. cuz the receiver side simply access the data.
- **Simple** IPC applications easy to implement
- Unsynchronized communication possible
- multi sender : multi receiver communication simple to implement

### ****Semaphore – Simple Interactions****

- mutual exclusion
    - make sure that only one entity can access the object protected by mutex at the same time: mutex or semaphore-like-mutex
    
    ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled.png)
    
- Unilateral synchronization(one side)  (especially suitable for producer / consumer problem)
    
    ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%201.png)
    
    - Problem 1: Producer and consumer access the shared object cause conflict
        
        Solution: add mutex to protect the shared object so that only one entity can access it at the same time
        
        ```cpp
        
        Sem mutex(1);
        
        void producer() {
        	mmutex.wait();
        	shared.put();
        	mutex.signal();
        	elem.signal();
        }
        
        void consumer() {
        //Q:what happend if we add	m.wait(); here(above elem.wait()?
        // it will cause deadlock.
        //if there is no ele in the queue, the consumer 
        //will firstly lock the mutex and then blocked at the line elem.wait(), which means 
        // the consumer waits for producer to insert element to the queue,
        //However producer() can never to put ele into the queue since the consume hold the
        // mutex
        // take away: wait in the mutex is not good idea, because it will cause deadlock
        
        	elem.wait();
        	mutex.wait();
        	shared.get();
        	mutex.signal();
        }
        ```
        
    - Problem 2: shared object has upper bound, insert element to a fulled  queue may cause it to crash
        
        add another semaphore to indicate the max size of the queue
        
        ```cpp
        Sem m(1);
        Sem free(42); //the queue has 42 free slot
        
        void producer() {
        	free.wait();
        	m.wait();
        	shared.put();
        	m.signal();
        	elem.signal();
        }
        
        void consumer() {
        
        	elem.wait();
        	m.wait();
        	shared.get();
        	m.signal();
        	free.signal();
        }
        ```
        
- Resource-oriented synchronization

![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%202.png)

- Readers—writers problem
    1. Q what is reader / writer problem?
        
        Writers need exclusive access to memory while multiple readers can work concurrently
        
    2. Design pattern
        
        ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%203.png)
        
    3. Sample code 
        - p represent wait, v represent signal
        - `w_mutex`: make only one writer in critical section
        - `read`, `wirte` make sure only writer or readers in critical section
        
        ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%204.png)
        
        - Why not `read.p` in CS?
            
            ```cpp
            sem read(0) //initilize with 0
            // Acquire (Reader)
            	mutex.p();  // mutex used to protect whole CS
            	ar++; // 
            	if (aw==0) { //no active writer
            		 rr++; 
            		 read.v(); 
            	}
            	mutex.v(); // leave cs
            	read.p(); //if there is active writer, active reader blocks here 
            
            Q: Why not read.p in CS Like this?
            sem read(0) //initilize with 0
            // Acquire (Reader)
            	mutex.p();  // mutex used to protect whole CS
            	ar++; // 
            	if (aw) { //no active writer
            		 rr++; 
            		 read.p(); 
            	}
            	mutex.v(); // leave cs
            // we will block within the CS and mutex will stay blocked.
            // none of the other operations will be able to continue
            ```
            
        - 这是 读者优先 还是 写者优先问题？
            - this is writers preference , because in writer’s acquire, it only waits for reading readers who currently reading data. However in reader’s acquire, it has to wait for active writer who is going to write data.
            - 读者优先:
                - active writer must for active reader, active reader only wait for writing writer
            - 写者优先
                - active writer only for reading reader, active reader must wait for active wirter
- ****Semaphore – Discussion****
    - Pro
        - very flexible to use
        - basic for implementing other synchronization tech.
        - Programming-languages support
    - Con
        - Add semaphore is trick, can cause errors and bugs

### ****Monitors – Synchronized ADTs [1]****

- Idea:  Couple abstract data type with synchronization properties
    - Only one entity can access any of these methods belonging to the monitor at the same time

![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%205.png)

- What happens if we use monitor for producer consumer problem
    
    ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%206.png)
    
    monitor  is not suitable for producer consumer problem, because dead lock may happens.
    
    if `consumer` blocked at `queueNotEmpty.wait();` , the dead lock will happen. Because it prevent producer to call the `produce`() method.
    
- Monitors is not suitable for Readers, Writers problem too, because only one entity can access any of these methods belonging to the monitor at the same time, which means monitor doesn’t support multiple readers access the data at the same time.
- Monitor implementation:
    
    ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%207.png)
    
- Monitors – Discussion
    - Limitation：
        - Limits concurrency to full mutual exclusion (Only one entity can access any of these methods belonging to the monitor at the same time)
        - Coupling of logical structure and synchronization not necessarily “natural”
            - see readers—writers example
    - Pro:
        - Suitable for use cases where mutual exclusion can be apllied
        - Programming-languages support
    - Conclusion: Synchronization should be separated from data organization and methods

### Path Expressions

- Idea:
    - Flexible expressions describe permitted sequences of execution and the object-access degree of concurrency
    - Note the idea here is that user only need to write functions/shared object and how they want those functions/ object to be accessed using the path expression. The compiler will then add correspondent synchronization primitive to the code to enforce the access patter of shared objects.
- Grammar:
    - **path** name1, name2, name3 **end**
        - Arbitrary order and arbitrarily concurrent execution of name1–3
    - **path** name1; name2 **end**
        - Before each execution of name2 at least once name1
    - **path** name1 + name2 **end**
        - Alternative execution: either name1 or name2
    - **path** N:(path expression) **end**
        - max. N control flows are permitted to be in path expression
- example: **path** 10:(1:(insert); 1:(remove)) **end**
    - Synchronization of a 10-element buffer
        - Mutual exclusion during execution of insert and remove
            - only 1 process can run insert
            - only 1 process can run remove
            - remove and insert can run concurrently, but each of them cannot overlap itself
        - At least one insert before each remove
        - Never more than 10 finalized insert operations
- Path Expressions – Discussion
    - Pro
        - More complex interaction patterns possible than with monitors
            - support reader writer problem: `read + 1: write` ==`read + （1: write`） + 代表 在一个时间点只有 reader 或 writer 可以 被执行， `1: write` 意思是每次 最多只有一个writer可以被执行
        - Compliance with interaction protocols is enforced by compiler
            - Less bugs!
    - Con
        - No support for path expressions in common programming languages
        - Synchronization of the state machine itself can become the bottleneck

## ****IPC via Messages****

- Use cases/Constraints
    - IPC across machine boundaries
    - Interaction of isolated processes
- Positive properties
    - Uniform paradigma for IPC with local and remote processes
    - Buffering and synchronization if necessary
    - Indirection allows for transparent protocol extensions
    - High-level language mechanisms such as OO messages or procedure calls can be mapped to IPC via messages (RPC, RMI)
- Message-based Communication: Variations of send() and receive()
    - synchronous / asynchronous (blocking / non-blocking)
    - buffered / not buffered
    - direct / indirect addressing
    - fixed / variable message sizes
    - symmetric / asymmetric communication
    - with / without timeout
    - broadcast / multicast

## ****Basic Abstractions in Operating Systems****

- Which basic IPC abstractions do operating systems offer?
    - UNIX: Sockets, System V Semaphore, messages, shared memory
    - Windows NT/2000/...: Shared memory, events, Semaphore, Mutant(mutex in user space, mutex can only been used in user space), sockets, asynchronous I/O, …
    - Mach: Messages to ports and shared memory (with copy-on-write)
- System-internal abstractions
    - Practically always: Semaphore
        - Mutual exclusion & unilateral synchronization → very common use cases
    - Microkernels and distributed operating systems: Messages
        - Basis for message implementations: Synchronization primitives
    - Monolithic systems: Semaphore and shared memory

## ****Duality of Concepts****

### Messages in Shared Memory

1. Mailbox:  use Semaphores + shared memory to build a mailbox-like messages system
    - Messages are not copied
        - ender provides memory
    - mailbox abstraction allows for M:N IPC
    - Receive may block
    
    ```cpp
    class Mailbox : public List {
     Semaphore mutex(1);
     Semaphore has_elem(0);
    public:
     void send(Message *msg) {
    	 mutex.p();
    	 enqueue(msg); // from List
    	 mutex.v();
    	 has_elem.v();
     }
     Message *receive() {
    	 has_elem.p();
    	 mutex.p();
    	 Message *result = dequeue(); // List
    	 mutex.v();
    	 return result;
     }
    };
    ```
    
2. Shared Memory with Messages base on page fault handling
    
    idea: Process A access the b page of the SVM, PFH figured the page is not present → PFH contact remote machine to get the page b, copies page b form right side to the left side. set right side invalid, and retry the access
    
    ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%208.png)
    
    ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%209.png)
    
3. ****Duality – Discussion SVM****
    - Distributed virtual shared memory allows …
        - to apply the multiprocessor programming model on distributed systems
        - IPC via (virtual) shared memory in spite of isolated address spaces
    - Problems
        - Communication and trap-handling latency
            - 2 Processes constantly access the a variable on page A, page A will be moved back and forth, which give OS a lot of overhead.
        - “False sharing” – Page size does not match object size
            - shared object size doesn’t match the page size, when process A try access object A on page A, page fault handler will map page A from process B to A. However, if object size is smaller than page size, other object may be copied to the process A as well, which process A doesn’t want to access.
    - Approaches
        - Weak consistency models, e.g.
            - Not every access causes a trap, accept not completely updated values
            - Distribute changes asynchronously via broadcast / multicast

### ****Duality – Active Objects****

- Suited for access synchronization in systems with message-based IPC
- Synchronous `send()` blocks a client as long as the server is still busy (as long as server doesn’t send the reply, the client will be block at send())
- Server side mutual exclusion guaranteed by the while loop, because the server can only process 1 request before it receive another req.

![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%2010.png)

- Reader writer problem can be solved use this model
    
    ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%2011.png)
    
    - Actual read/write operations happen concurrently in a child process
        - Server doesn’t have to wait for expensive read/ write op —→ speed up the server
        - Allow multi reader to read cuncurrently
        
        ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%2012.png)
        
    - 2 primary changes compared to the reader writer implementation using semaphore
        - no protect mutex  anymore, mutex exclusion is guaranteed in the server
        - add message queue read and write
            - if there is active writer, we simply put read request into the read queue, once the writing is done, we dequeue the read and send reply to client, which indicate that the client is allowed to send a `read_msg` to read the data
    
    ![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%2013.png)
    

### Duality – Discussion

- Is there a **fundamental difference** between IPC via shared memory and IPC via messages?
    
    No,  From the synchronization and concurrency perspective, they are identical
    
    Example: Reader—writer monitor vs. server:
    
    - Monitor: 2 potential waiting points
        - Client is delayed for mutual exclusion
        - Client is potentially further delayed due to a condition variable
    - Server: 2 potential waiting points
        - Reply is delayed because the server serves other requests
        - Reply is potentially further delayed if the request must be enqueued in a waiting queue

![Untitled](Week10%20Inter-Process%20Communication%20(IPC)%20117680ccc65242448bbe9d6cf248387f/Untitled%2014.png)
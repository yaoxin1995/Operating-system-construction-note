# exercise-5

- Course: OSC lab
- [Material](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/E05-Preemption.pdf)
- [video](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/E05-Preemption.mp4)
- [task-4](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/tasks/task4/)     [task-5](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/tasks/task5/)

# Lab Task #4: Tips & Tricks

## 

## Lab Task #4: Overview

### 

![Untitled](exercise-5%2054fa68c9a9db4c07a412bd33191e9f8c/Untitled.png)

- main work:  toc doing actual context switching and preapre  1. couroutine for excution

## Part a) Coroutine

![Untitled](exercise-5%2054fa68c9a9db4c07a412bd33191e9f8c/Untitled%201.png)

- each appl must be implemented as a couroutin
    - Con : each appl must know all other appls and has to explicitly name the next coroutine to switch to
- Cooperative Thread Switch
    
    ![Untitled](exercise-5%2054fa68c9a9db4c07a412bd33191e9f8c/Untitled%202.png)
    
- Functions
    - **Coroutine (void* tos);**
        - In the coroutine constructor, the register values are initialized so that the stack pointer initially points to tos and on first activation execution begins with the kickoff function.
    - **void go ();**
        - This method is used for the first activation of the first coroutine in the system. Therefore no register values must be saved here.
    - **void resume (Coroutine& next)**
        - This method triggers a coroutine switch.
    - **virtual void action () = 0;**
        - = 0 mean pure virtual, the base class doesn’t has any implementation, derive class has to implemente the function.
        - The method action represents the actual job of the coroutine.

### **toc : top of stack**

- Call from C++ necessitates declaration with extern "C"!
- Functions
    - **Prepares the struct toc for the first activation.**
    
    ```cpp
    void toc_settle (struct toc* regs, 
    void* tos,void (*kickoff)(void*, void*, …),
    void *coroutine);
    //tos : top of stack
    
    //creat a stack 
    char stack1[65536];
    //- how to pass tos pointer?
    //- Couroutine constructor only saves a tos pointer, so if we
    //  create a coroutine, we have a top of stack.
    Coroutine c1(
    	stack1 + sizeof(stack1)            
    );
    
    void toc_settle (struct toc* regs, 
    void* tos,void (*kickoff)(void*, void*, …),
    void *coroutine){			
    //- tos : is that be passed to the couroutine constructor,
    // the couroutine constructor calls toc_settle() and passes the 
    // TOS pointer on  
    // in toc_settle we have to prepare the stack for the first 
    // couroutine activation
    
    //- how to put the cotourine pointer into the stack?
    //- make tos pointer point to void.
    
    //char *p; //0x1000
    //p++;     //0x1001
    //int *p2  //0x2000
    //p2++     //0x2008,32bit interger
    
    //void *p3
    //p3++;        //not standard cuz dk the size of void
    
    //pointer casts the p to void **.
    void **p = (void **)tos
    p--;
    //p is a pointer which point to a pointer point to void
    // decreases the stack pointer by the size of a pointer
    *p = coroutine; 
    
    //dereference a ** what we can get is the mem addr where i can store and read a void pointer
    }
    
    ```
    
    - Loads the non-volatile processor registers with the contents of the struct regs.
        
        ```cpp
        void toc_go (struct toc* regs);
        ```
        
    
    ```cpp
    
    void toc_switch (struct toc* regs_now, struct toc* regs_then);
    ```
    
    ![Untitled](exercise-5%2054fa68c9a9db4c07a412bd33191e9f8c/Untitled%203.png)
    
    - Performs a context switch. To do this, the current register values in regs_now must be saved and replaced by the values of regs_then.
        - what has toc_switch done?
            1. saves the NV-R of the callee coroutine to its toc
            2. loads NV-R form another toc including stack pointer and this pointer sp point to new stack
            3.  returns to the top value of stack
        - and loads NV-R form another toc including stack pointer(at this point sp point to new stack), and then it returns.

## Part b) Dispatcher

![Untitled](exercise-5%2054fa68c9a9db4c07a412bd33191e9f8c/Untitled%204.png)

- konws which coroutine is currently active(life pointer)
- now we know which couroutine is currently active, the next couroutine to be run will be provided by the scheduler

### Scheduler

- Considers the set of ready threads
- The currently running thread is always also affected by the decision
- We pass the selected, new thread to the dispatcher.

### Dispatcher

- The dispatcher manages the life pointer, which indicates the currently active coroutine, and performs process switches.
    - **Dispatcher ()**
        - The constructor initializes the life pointer with null to indicate that no coroutine is known yet
    - **void go (Coroutine& first)**
        - With this method the coroutine first is put in the life pointer and started
    - **void dispatch (Coroutine& next)**
        - This method sets the life pointer to next and performs a coroutine switch from the old to the new life pointer
    - **Coroutine* active ()**
        - This can be used to determine which coroutine is currently in control of the processor.

## Part c) Cooperative Scheduling

- [use static_cast<Entrant *> to protect multiple inheritance class.](https://zhuanlan.zhihu.com/p/37445001)
- The scheduler manages the ready list (a private Queue member of this class), which is the list of processes of type Entrant that are ready to run. The list is processed from front to back. New processes, and those that yield the processor, are appended to the end of the list.
    - **void ready (Entrant& that)**
        - This method registers the process that with the scheduler. It is appended to the end of the ready list.
    - **void schedule ()**
        - This method starts up scheduling by removing the first process from the ready list and activating it
    - **void exit ()**
        - With this method a process can terminate itself. The scheduler does not append it again to the end of the ready list. Instead, it removes the first process from the ready list and activates it.
    - **void kill (Entrant& that)**
        - With this method a process can terminate another one (that). The process that is simply removed from the ready list and is thereby never scheduled again.
    - **void resume ()**
        - This method allows to trigger a context switch without the calling Entrant having to know which other Entrant objects exist in the system, and which of these should be activated. This decision is made by the scheduler using the entries in its ready list. In this system, it shall append the currently running process to the end of the ready list and activate the first one.

## ****Class Entrant****

- **Entrant (void* tos);**
    - The Entrant constructor passes only the tosparameter to the Coroutine constructor.

## Questions:

- [Scheduler::kill](https://github.com/yaoxin1995/oostubs/blob/6433e507db087fcf51c391aabd85db07b4cd018e/thread/scheduler.cc#L39)
    - kill should also can kill the thread itself.
- [main](https://github.com/yaoxin1995/oostubs/blob/6433e507db087fcf51c391aabd85db07b4cd018e/main.cc#L127)
    - appl should also been put into ready list
- ****[kickoff.cc](https://github.com/yaoxin1995/oostubs/blob/6433e507db087fcf51c391aabd85db07b4cd018e/thread/kickoff.cc#L29)****
    - scheduler.exit() instead infinite loop
    

# Lab Task #5: Overview

## Time slice Scheduler

- Goal: protect critical section OS using the prolog/epilogue model
- Scheduler : Timer interrupts trigger thread preemption
- Calling guard-protected methods of the scheduler:
    - Scheduler variable is now change to Guarded_Scheduler

## Preemptive Thread Switch

![Untitled](exercise-5%2054fa68c9a9db4c07a412bd33191e9f8c/Untitled%205.png)

## Thread Switch in epilogue

- Tips
    - Never call enter from the epilogue (double request)
    - Basic rule (see above) also holds for the first thread activation(!)  
        - in main : schedule() doesn’t  have secure to be protected. so before we activate the first thread we need to use a guard lock
        - kickoff is not directly called by any method, it actually was jumped in. so kickoff function is expcected to be called with gaurd locked.  before this thread is running we need to make sure to release the guard lock.
        - **summary** : every require needs some matching release.
    - 

## Class Guarded_Scheduler

- C++ detail:
    - Because methods of Guarded_Scheduler have the same names asthose of the base class Scheduler, they hide them
    - Access hidden methods: explicitly provide **base-class scope** when calling a method
    
    ```cpp
    Guarded_Scheduler scheduler;
    Application appl1, appl2;
    scheduler.ready (appl1); // Guarded_Scheduler method
    scheduler.Scheduler::ready (appl2); // Scheduler method
    ```
    
- Programming the 8254
    
    // |   bits   |   value  |  meaning
    // |    0      |     0      |  Binary counting of 16 bits
    // |    1-3   |    010   |  Periodic interrupt
    // |    4-5   |     11   |  Low-order, then high-order counte byte
    // |    6-7   |     00   |  Counter 0
    // 7 6 5 4 3 2 1 0
    // 0 0 1 1 0 1 0 0  = 52 = 0x34
    

# PIT Programming

# Preemptive Scheduling

- Thread A uses the CPU for 18 ms and then voluntarily calls resume()
- Thread B continuously uses the CPU and never voluntarily yields

![Untitled](exercise-5%2054fa68c9a9db4c07a412bd33191e9f8c/Untitled%206.png)

- solution :
    - reset the timer at first resume(): Thread B get full time slice.
    - RR or VRR

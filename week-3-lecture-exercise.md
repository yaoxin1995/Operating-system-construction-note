# week-3

- Course: OSC
- Materials: [https://os.inf.tu-dresden.de/Studium/OSC/SS2022/L03-IRQ-HW.pdf](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/L03-IRQ-HW.pdf)

# Lecture

## Interrupt

- why we need Interrupt?
    - Overlapped I/O: meant that multiple I/O transfers (normally each I/O to a different device) could occur at the same time (like concurrent reads from tape and writes to disk).
    - Time sharing:

## Prioritization mechanism

- decided which interrut is more 'important', when multi interrupt were signaled at once.
- Solution
    - SW: one IRQ (interrupt request) line and queries/services devices in a specific order.
    - HW: using a crcuit to assign priorities to devices and handling the most urgent request.
    - static priorities
    - dynamic priorities

## Lost interrupts

- why would INTERRUPT LOST happend?
    - The CPU cannot handle new interrupts, wenn interrupts are disabled, and the Memory for IRQ is NOT enough for all interrupts.
- Solution here:
    - saving Memory space by making short interrupt handler routine as short as possible.
    - The interrupt should not be disabled for longer than the CPU needs.
    - wenn an interrupts signals more than one I/O operation, the device driver should do sth to solve.

## Interrupt Dispatch

- Interrupt dispatching is a mechanism wherein the CPU passes execution control to software to decide which device is triggered the inerrupt.
- Problem here:
    - time comsuming
    - state of I/O busses and other device might be modified.
- Solution
    - Interruppt vector : [https://en.wikipedia.org/wiki/Interrupt_vector_table](https://en.wikipedia.org/wiki/Interrupt_vector_table)
    - The system program must maintain an interrupt vector table, and each table entry records the address of an interrupt service program (ISR).
    - When an external event or exception is generated, the hardware will generate an interrupt flag. The CPU can based on the interrupt flag to get the interrupt vector number, queries the interrupt vector table (based on the address and interrupt vector number of the interrupt vector table) to obtain the address of the interrupt program for the corresponding interrupt number, and further executes the corresponding interrupt handler.

## Saving State

- after all the handler routine return to the point where the program stopped
- Transparency: Interrupt should not be noticed by the ctrl flow, otherwise any ctrl flow can be interrupted by the interrrupt hanler. besides time other states should stay the same.
- Solution:
    - HW
        - saving the state (return adr & status register)
        - IRET(interrupt return), RTE:
    - SW

## Nested Interrupt Handling

- INTR can be also interrrupted by other INTR with higher priority. This enables the szstem to prioritize interrupts and reduce the latency of high priority events. which means INTR should be interruptible.
- avoid unlimited nesting --> stack may be overflow....
- Solution:
    - saving current priority of INTR in status reg, previous prior. on a stack

## Multiprocessor Systems

- Problem :
    - decided which CPU should be used to handle current INTR when the system is Multiprocessor system.
    - Inter-processor interrupts (IPIs): In multi-processor systems, the operating system needs to coordinate operations among multiple processors.
- Solution:
    - More complex interrupt-handling hardware for multiprocessors
        - static destination
        - random destination
        - programmable destination
        - destination depending on current CPU load
        … and combinations thereof.

## Hazard

### Spurious Interrupts

- Interrupt-handling mechanism can be presented with spurious* interrupts, caused e.g. by Hardware errors or incorrectly programmed devices
- Solution: Programming the operating system to predict Spurious Interrupts

### Interrupt Storms

- thrashing: [https://en.wikipedia.org/wiki/Thrashing_(computer_science)](https://en.wikipedia.org/wiki/Thrashing_(computer_science))
- High interrupt frequency can overload or “freeze” a computer
- solution : Detect interrupt storms or Deactivate culprit device
# week-2

- Course: OSC
- [Materials](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/L02-Development.pdf)
- [Video](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/L02-Development.mp4)

# Lecture: Operating-System Development 101

# How to get your OS onto the target hardware?

## Compilation/Linking

- Complication, Assembly and Linking: [https://www.cnblogs.com/glacierh/p/4678229.html](https://www.cnblogs.com/glacierh/p/4678229.html)
    
    ![Untitled](week-2%20909c5cb9320f4c7d9dbabc4ba44f133f/Untitled.png)
    
- why program can not run on bare metal?
    1. no dynamic linker avalibale
        
        →we donot have “loader” on bare metal machine. 
        
        what if we statically link the programm?
        
        - the standard c++ lib that we statically linked requires runtime environment around. We cannot use regular C/C++ runtime libraries→ so we donot have systemcall
    2. no syscall → we cannot use regular std c++ lib
    3. Generated addresses refer to virtual memory, we donnot know if there’s memory mapped to this virtuell address. 
        - set a page table
    4. High-level language code excepts environment been setted in some way. (e.g direction Flag is cleared, which register contains function return value etc.)
        
        →Own startup code (written in assembler) must prepare high-level language code execution.
        
- Is OS development in a high-level programming language possible at all?
    - yes but we can *not* use standard lib of c++

## Booting

- idea : start a complex system using a simple system.

### Process

1. system start
2. Jump (via RESET vector or directly) to fixed address(jumps to BIOS)
3. BIOS or firmware start/ROM OS start
    1. frimware provide 
        1. boot process
        2. a sort of basic OS
            1. there are some BIOS functions u can call e. g read a block  from a block device
4. Copy data segments, initialize BSS segment
    1. the BIOS live in ROM. In order to have variables writable, the copy data has to transfer to RAM. 
5. Hardware init(count how much mem is avaiable) and test(write bits in mem and check back if they are the same)
    - error →system stop
    - otherwise →Search for boot medium(configurable), the system on the boot medium is loaded and started
        - error →system stop
        - OS loaded → DONE
    - OS came from ROM→DONE

### Boot Sector

![Untitled](week-2%20909c5cb9320f4c7d9dbabc4ba44f133f/Untitled%201.png)

- if 0xaa55 is in this position present, these 512 bytes will be copied into  0x7c00 and BIOS jumps there

### Boot loader :used by ooStubs

- system-specific boot loaders
    - Define hardware/software state
    - opt. Load further blocks with boot-loader code
    - Pinpoint the actual system on the boot media
    - Load the system (via BIOS functions)
    - Jump into loaded system
- Boot loader on disks not flagged as “bootable”
    - Error message, halt / reboot
- Boot loader with boot menu (e.g., GRUB)
    - Display a menu
    - Emulate BIOS when booting the selected system
    (load boot block to 0x7c00, jump)

## Testing and Debugging

### “printf Debugging”

- you need to has your own printf and stuff
- printf() often changes the debuggee’s behavior
    - Problem vanishes / changes symptoms
    - Unfortunately particularly true for OS development
- similar behaviour: LED, Serial interface

### (Software) Emulators

- Emulate real hardware in software
    - + Simplifies debugging
    - + Shorter development cycles
    - - Emulator and real hardware may differ in details!
    - - Harder to find bugs in a complete system than during incremental development
- Emulation: a special case of virtualization
Provides a virtual resource Y (e.g., an Arm CPU)
based on a resource X (e.g., the systems x86-64 host CPU
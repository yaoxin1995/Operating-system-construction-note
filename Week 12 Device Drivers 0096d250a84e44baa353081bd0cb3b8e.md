# Week 12 Device Drivers

Created: August 22, 2022 3:54 PM
Reviewed: No

I/O subsystem design requires much expertise

- As much reusable functionality as possible in driver infrastructure
- Well-defined driver structure, behavior and interfaces, i.e. a driver
mode

# Requirements for OS / driver design

In general, we want the OS / driver to have following features:

- resource efficient:
    - Work fast
    - Save energy
    - Save memory, ports, interrupt vectors
- have **Uniform access mechanism**
    
    → cuz appl in the end wants to interact with like network-card don#t want to have diffrent implementation for different kind of network-card
    
    - **Minimal set of operations** for different device types
        - network-card : read, set the networking adr and send,receive package
    - **Powerful operations** for diverse application types
- **Also support device-specific access functions (ioctl)**
- Activation and deactivation at runtime (hot plugin)
- Generic power-management interface

## Linux – Uniform Access (1)

1. How to access the device?
    
    Devices are accessible via names in the file system
    
    - Pros:
        - System calls for file access (open, read, write, close) can be used for accessing variant device driver
        - Access permissions can be controlled via **file-system mechanisms**
        - Applications see no difference between files and “device files”
    - Problems
        - Block-oriented devices must be adapted to byte stream
        - Some devices hardly fit this schema
            - Example: 3D graphics adapter
2. The different mode for accessing the device diver
    - Blocking input/output (default case)
        - read: Process blocks until requested data is available
        - write: Process blocks until writing is possible
    - Non-blocking input/output
        - instead of blocking, read and write return -EAGAIN
        - Caller may/must repeat the operation later
        - open/read/write with additional flag O_NONBLOCK
    - Asynchronous input/output (not cause the requesting process to be blocked)
        - these functions work by telling the kernel to start the operation and to notify us when the entire operation (including the copy of the data from the kernel to our buffer) is
        complete
        - AIO, IO_URING, etc.

## Windows – Uniform Access

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled.png)

1. How to access the device?
    
    Devices are Executive kernel objects. If a win32 application wants to access the device, it can issue a request(createfile, readfile, writefile) to Win32 subsystem with the parameter such as device name, and in which directory the device resides (this information is stored in Win32 PCB, process control block), for example, here win32 app issue a request to COM1 (representing the serial device), and send the device name (COM1) and the device location (/GLOBAL) along with the request. The win32 forward the request to the NT kernel since devices are all executive kernel objects. In NT kernel, devices are managed by a tree-like structure. Each node in the tree contains a 1 to 1 mapping from the device name to the real device. When the request comes in, the NT kernel uses the device location (/Global) to find the subtree where the device resides, then it uses the device name specified in the request to find the real device.
    
2. Synchronous or asynchronous input/output mode for accessing the device possible
    1. Default is asynchronous  mode
3. What is the difference between the windows and Linux uniform access?
    
    In general, they are very similar, (the all have have name, path that can be traversed to access these devices). the difference is that in windows the device tree is in the kernel and not visible in the file system. However, linux has multiple pseudo-filesystem which provides an interface to kernel data structures, including the device tree  
    

## Linux – Device-specific Functions

1. Special device properties are (classically) controlled via `ioctl`
2. The `sysfs` filesystem is a pseudo-filesystem which provides an interface to kernel data structures.  (More precisely, the files  and directories in `sysfs` provide a view of the `kobject` structures defined internally within the kernel.)   The files under `sysfs` provide information about devices, kernel modules, etc,. User can use it to read device information and control device in variant way
    
    ![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%201.png)
    

## Windows – Device-specific Functions

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%202.png)

# I/O-System Structure

In general, App call the Device independent IO function such as read/write system call to send the IO request to driver. Driver then forward the request to the hardware access functions, which is responsible for communicating with real device.

### 2 type of IO system structure:

1. Drivers with different interfaces
    
    ![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%203.png)
    
    Pro: allow to fully utilize all device properties
    
    Con: necessitate extending the I/O system for each driver
    
    - Large variety of devices→ high efforts
    - Unrealistic:  too many work to do
2. Drivers with a uniform interface
    
    ![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%204.png)
    
    Pro: 
    
    - enable a (dynamically) extensible I/O system
    - allow flexibly “stacking” device drivers, raid-like device driver
        - Virtual devices, like vbdev in spdk
        - Filters

### driver model

- A list of expected driver functions
    - what function call be called from the top or buttom, e.g intr coming form the hardware also necessitate callback.
- Definition of optional and mandatory functions
- Functions the driver may use
- Interaction protocols
    - how to initialize hardware before use
    - How to transform data from user to kernel (copy_from_user) etc
- Synchronization schema and functions
    - if driver touches kernel state, make sure the consistency is maintained
- Definition of driver classes if multiple interface types are inevitable
    - driver class serial communication device
    - driver class sound card
    - driver class networking card
    - driver class block device

# Device Drivers and Environment

### ****Device-driver Requirements****

- A central requirments of devicer driver is that it must be able to manage multiple devices
- Allow assigning device files
- Management of multiple device instances
- Operations:
    - Hardware detection
    - Initialization and termination
    - Reading and writing of data
        - possibly [scatter/gather](https://en.wikipedia.org/wiki/Vectored_I/O)
    - Control operations and device status
        - e.g. via ioctl or virtual file system
    - Power management
- Internal challenges:
    - Synchronization
    - Buffering
    - Requesting needed system resources

## Linux Solution

### ****Linux – Driver Template: Operations****

Linux has a `[file_operation`](https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L1964) struct, which define the function signature the user can use to interact with the device. It is the device responsibility to implement the functions. Below, the driver has implement open, release, read function in the `file_operation` struct. During the driver initialization, the `file_operation` struct will be registered to Linux driver infrastructure, so that the user can use open, read, system call to intterract with the device.

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%205.png)

### Linux – Driver Template: Registration

every driver module has to have a initialization and a exit function, where the driver can allocate / register itself and free the resources.

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%206.png)

### ****Linux – Driver Infrastructure****

Driver programmer don’t have to implement every thing by their self.  A bunch of functions that can be used by all driver have been implemented in kernel (driver infrastructure), and driver can use them  directly. For example:

- Allocate and release resources
    - Memory, ports, IRQ vectors, DMA channels
- Hardware access
    - Read and write ports and memory blocks
- Dynamically allocate memory
    - for buffers foe example
- Blocking and waking processes
    - Wait queues
- Registering interrupt handlers
    - Low-level → prolog
    - Tasklets for longer activities → more or less epilog
        - runs on software intr context L1/2
- Special APIs for different driver classes
    - Character devices, block devices, USB devices, network interface cards
- Integration in proc or sys file system

## Windows solution

### Windows – I/O System(略)

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%207.png)

- **WMI(windows management instrumentation)**: a uniform way to instrument and monitor events in the kernel and monitor perfermance
- **PNP(plug and play**): e. g plug in a USB → pnp manager talks to the user mode part that pops up some kind of dialogue to asks u to insert a driver CD, 现在都是自动在微软下载。
- this driver comes with a **setup components**:
    - **.inf** (driver installation file): matadata e.g
        - which devices this driver for
        - what properities does this driver have
        - which file s are part of this driver
        - which registry entries have to be entered to the registry
        - which IDs the driver have (vendor id, device id)
    - **.cat(catalog file that primarily holds hashed over all files that comes with driver)**
        - the cat file is usually cryptographically signed, so u can make sure that the device vendor is authentic.
- registry:
    - can be accessed by the I/O manager,
- I/O manager controls input and output using the drivers here

### ****Windows – Driver Structure****

- The I/O system controls the driver using the …
    - Initialization/unload routine
        - called after/before loading/unloading the driver
    - Routine for adding devices
        - PnP manager found a new devices for the driver
    - Dispatch routines
        - Open, close, read, write, and device-specific operations
    - Interrupt Service Routine
        - called from the central interrupt dispatch routine →prologue
    - DPC routine (deferred procedure call)
        - Interrupt-handling “epilogue”
    - I/O completion and cancel routines
        - Information on the status of forwarded I/O jobs

### Windows – Typical I/O Procedure (Q)

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%208.png)

When a system call comes in, in this case, the `NtwriteFile`, the IO manager first finds the file system and responsible driver from the file handler. Then it asks the file system driver to write the data stored in `char_buffer` to an offset in the file. However, the file system driver doesn’t talk to the disk directly. Instead. it translates the offset to the offset to disk position and issues another IO request to IO Manager. IO manager then forwards the request to the raid disk driver. Note that in the figure, we have a raid driver system, on top, we have a virtual driver, which manage the 2 disk driver below, and is responsible for receiving the request from other components in the system. The 2 disk drivers blow charge to talking to 2 disks, respectively. After the raid disk gets the request, it translates the disk position to disk number and block number, then forwards the IO request to the correspondent driver, in this case, the IO request is forwarded to Disk driver 2. The disk driver 2 then uses the function provided by driver infrastructure to notify that the data in the `char_buffer` need to write to disk. Then, instead of waiting for the IO completion, it returns to the app immediately, since windows use the asynchronous IO.

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%209.png)

When the disk finishes the write request, the disk controller signals the disk driver the IO completion via an interrupt. Then IO manage calls the interrupt handler routine in the disk driver 3 to handle the interrupt. With the help of the **IRP stack,** the IO manager knows that it has to first trigger a completion routine in the raid driver and then a completion routine in the file system driver in order to finish the asynchronous IO request. Depending on the IO request complexity, the file system driver may issue multiple sub-requests to the disk driver, for example, to flush the kernel buffer to disk, etc,.

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%2010.png)

# Summary

![Untitled](Week%2012%20Device%20Drivers%200096d250a84e44baa353081bd0cb3b8e/Untitled%2011.png)
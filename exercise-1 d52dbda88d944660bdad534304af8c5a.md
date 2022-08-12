# exercise-1

- Course: OSC lab
- [Materials](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/E01-Cpp-CGA.pdf)
- V[ideo](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/E01-Cpp-CGA.mp4)
- [task-1](https://os.inf.tu-dresden.de/Studium/OSC/SS2022/tasks/task1/)

# C++ crash course (Part 1)

## Control Structures and Variable Types

- Conditional statements, loops, compound statements (blocks)
    - are identical in C++ and Java! (ignoring variants in recent C++ versions)
- C++ allows “global” functions, while in Java methods must be part of a class.
    - In particular, C++ allows calling “normal” C and assembler functions
    - and you can make C++ functions callable from C and assembler via extern "C"
    - One example for an important global function is main()

# CGA programming

### Output Stream

![Untitled](exercise-1%20d52dbda88d944660bdad534304af8c5a/Untitled.png)

- CGA_Stream is actually the output device. we want to our appl to be able to write text

- Stringbuffer: put(c), flush()
    - Why buffer? Reasonable buffer size?
        - for performance reasons cuz the writing operation to device may involve some overhead that only to paid once only when the output data
- O_Stream: similar to C++ std::ostream
    - responsible for text handling
    - Formatting, number output
    - uses Stringbuffer::put(c)
- CGA_Stream::flush()

### CGA_Screen

- used by CGA_Stream during flush()
- show(x,y,c,attrib)
– Character c with attribute attrib at position x/y
– Code from the C++ crash course:

```cpp
char *CGA_START = (char *)0xb8000;
char *pos;
int x = 20, y = 20;
pos = CGA_START + 2*(x + y*80);
*pos = 'Q';
```

- missing : attrib

![Untitled](exercise-1%20d52dbda88d944660bdad534304af8c5a/Untitled%201.png)

- simply reuse the attribute that was in video mem at that position before.

- setpos/getpos
    - Change internal state of CGA_Screen
        - Current position needed in print()!
    - Position the CGA cursor
- In general: Access to PC devices
    - Two address spaces: Memory address space, I/O address space
    - Memory: addressable directly via pointers (video memory)
    - I/O: via CPU instructions in/out (inb/inw/inl; outb/outw/outl)
        - OOStuBS: encapsulated in class IO_Port
    - Some devices use both (e.g. CGA)
- CGA: Memory and I/O address spaces
    - Video memory mapped into memory address space
    - CGA registers mapped to I/O address space
- but: More registers than I/O addresses
    - Multiplexing via index/data ports
- **print(char *text, int length, unsigned char attrib)**
    - Uses show() and setpos()
    - Arrived at screen bottom? Scrolling! (How?)
        - scrolling :copy everthing 1 line up and clean up the last line and print stuff
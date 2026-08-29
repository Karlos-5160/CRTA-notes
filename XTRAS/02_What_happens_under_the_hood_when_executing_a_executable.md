When you type:

```bash
./program
```

a lot happens behind the scenes. Let's follow the journey from **your terminal → shell → kernel → program running in memory**.

---

# 1. Your shell receives the command

Suppose you're using Bash:

```bash
./program
```

The shell first parses what you typed.

The `./` means:

```text
.  = current directory
/  = inside that directory
```

So:

```bash
./program
```

means:

> Execute the file named `program` from the current directory.

This is different from:

```bash
program
```

Without `./`, the shell searches for `program` in directories listed in your `$PATH`.

For example:

```bash
echo $PATH
```

might show:

```text
/usr/local/bin:/usr/bin:/bin
```

The shell checks those directories.

But `./program` explicitly tells it:

```text
current directory
      │
      ▼
./program
```

---

# 2. The shell checks the file

The shell tries to access the file:

```text
./program
```

Linux checks things such as:

* Does the file exist?
* Do you have permission to execute it?
* Is it a directory?
* What type of executable is it?

For example:

```bash
ls -l program
```

You might see:

```text
-rwxr-xr-x 1 user user 12345 program
```

The important part is:

```text
x
```

The `x` means execute permission.

If the execute bit isn't set, you may get:

```text
Permission denied
```

You can add it using:

```bash
chmod +x program
```

---

# 3. The shell asks the kernel to execute it

The shell cannot directly load your program into RAM.

Instead, it makes a system call to the Linux kernel.

Conceptually:

```text
Terminal
   │
   ▼
Shell (bash/zsh)
   │
   │ execve()
   ▼
Linux Kernel
```

The important system call is:

```c
execve()
```

Conceptually, something like:

```c
execve("./program", arguments, environment);
```

The shell is basically telling the kernel:

> "Replace this process with this new program."

---

# 4. The kernel examines the file

Now the Linux kernel opens the file and checks:

> What kind of executable is this?

It could be:

### ELF executable

```text
ELF
```

### A script

```bash
#!/bin/bash
```

### A Python script

```python
#!/usr/bin/python3
```

### Or another supported executable format.

For a compiled C/C++ program, it will commonly be an **ELF file**.

You can check using:

```bash
file ./program
```

Example:

```text
ELF 64-bit LSB pie executable, x86-64
```

---

# 5. The kernel reads the ELF file

ELF stands for:

> **Executable and Linkable Format**

An ELF executable contains structured information describing how the program should be loaded.

Very simplified:

```text
+----------------------+
| ELF Header           |
+----------------------+
| Program Headers      |
+----------------------+
| .text                |
| Machine instructions |
+----------------------+
| .data                |
| Initialized data     |
+----------------------+
| .rodata              |
| Read-only data       |
+----------------------+
| Other sections       |
+----------------------+
```

The kernel reads the ELF headers to understand things like:

* CPU architecture
* 32-bit or 64-bit
* Where different parts should be mapped
* Entry point of the program
* Whether a dynamic linker is needed

---

# 6. Linux creates the program's memory space

The kernel creates a new **virtual address space** for the process.

A simplified process memory layout looks like:

```text
High Memory
+-------------------+
| Stack             |
| ↓                 |
+-------------------+
|                   |
|                   |
| Free space        |
|                   |
+-------------------+
| ↑ Heap            |
+-------------------+
| .bss              |
+-------------------+
| .data             |
+-------------------+
| .text             |
+-------------------+
Low Memory
```

### `.text`

Contains the actual CPU instructions.

For example, machine code like:

```text
MOV
ADD
CALL
RET
```

### `.data`

Contains initialized global/static variables.

Example:

```c
int x = 10;
```

### `.bss`

Contains uninitialized global/static variables.

Example:

```c
int x;
```

### Heap

Used for dynamic memory allocation:

```c
malloc()
```

### Stack

Used for:

* Function calls
* Local variables
* Return addresses
* Function arguments in some cases

---

# 7. What if the program uses shared libraries?

Most programs are **dynamically linked**.

For example:

```c
printf("Hello\n");
```

Your program may need functions from libraries such as:

```text
libc.so
```

You can inspect dependencies with:

```bash
ldd ./program
```

Example:

```text
linux-vdso.so.1
libc.so.6
/lib64/ld-linux-x86-64.so.2
```

At this point, the kernel may load the **dynamic linker/loader**.

For example:

```text
ld-linux-x86-64.so.2
```

The flow becomes:

```text
Your Program
     │
     ▼
Dynamic Linker
     │
     ├── loads libc
     ├── loads other libraries
     ├── resolves symbols
     │
     ▼
Program can start
```

So if your program calls:

```c
printf()
```

the dynamic linker helps connect that function call to the appropriate implementation in a shared library.

---

# 8. Arguments and environment variables are placed on the stack

Suppose you run:

```bash
./program hello world
```

The program receives:

```text
argv[0] = "./program"
argv[1] = "hello"
argv[2] = "world"
```

If your program is written in C:

```c
int main(int argc, char *argv[])
```

Then:

```text
argc = 3
```

The environment is also passed.

For example:

```bash
echo $HOME
echo $PATH
```

These become part of the environment available to the new process.

---

# 9. The dynamic linker finishes preparing the program

Before your `main()` function runs, some initialization happens.

A simplified flow is:

```text
Kernel loads program
        │
        ▼
Dynamic linker starts
        │
        ├── Load shared libraries
        │
        ├── Resolve required symbols
        │
        ├── Perform initialization
        │
        ▼
Runtime startup code
        │
        ▼
main()
```

An important point:

## The CPU usually does not jump directly into your `main()` function.

Instead, it starts at the executable's **entry point**.

The startup code eventually prepares the environment and then calls:

```c
main()
```

---

# 10. Finally, your CPU executes your program

Suppose your program is:

```c
#include <stdio.h>

int main() {
    printf("Hello\n");
    return 0;
}
```

Eventually:

```text
CPU Instruction Pointer
          │
          ▼
      main()
          │
          ▼
     printf()
          │
          ▼
      "Hello"
```

The CPU executes the machine instructions one by one.

Very simplified:

```text
Fetch instruction
       ↓
Decode instruction
       ↓
Execute instruction
       ↓
Next instruction
```

---

# 11. The program finishes

When `main()` returns:

```c
return 0;
```

the program eventually terminates through the runtime/kernel termination path, conceptually ending with something like:

```text
exit()
```

The kernel then:

* Marks the process as terminated.
* Records its exit status.
* Releases its resources.
* Closes remaining file descriptors.
* Reclaims the process's memory mappings when appropriate.

Your shell receives the result and can store the exit code.

You can check it with:

```bash
echo $?
```

For example:

```text
0
```

Usually means successful execution.

---

# The complete picture

When you type:

```bash
./program
```

this is the simplified chain:

```text
You
 │
 │ type ./program
 ▼
Terminal
 │
 ▼
Shell (bash/zsh)
 │
 │ finds ./program
 │
 ▼
execve()
 │
 ▼
Linux Kernel
 │
 ├── Check permissions
 ├── Identify executable format
 ├── Read ELF headers
 ├── Create process address space
 ├── Map executable into memory
 │
 ▼
Dynamic Linker (if dynamically linked)
 │
 ├── Load libc.so
 ├── Load other shared libraries
 ├── Resolve symbols
 │
 ▼
Runtime startup code
 │
 ▼
main()
 │
 ▼
CPU executes instructions
 │
 ▼
Program finishes
 │
 ▼
exit status → Shell
```

## One very useful cybersecurity perspective

You can actually **observe this process** using Linux tools:

```bash
strace ./program
```

This shows many system calls made while the program starts and runs.

For example, you may see:

```text
execve("./program", ["./program"], ...) = 0
```

You can inspect the ELF structure with:

```bash
readelf -h ./program
```

Program headers:

```bash
readelf -l ./program
```

Shared libraries:

```bash
ldd ./program
```

And the entry point can be inspected with:

```bash
readelf -h ./program | grep Entry
```

The next really interesting step is understanding **how the ELF loader, virtual memory, stack, heap, shared libraries, and the dynamic linker work together**, because that connects directly to Linux binary analysis and cybersecurity.

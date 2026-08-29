The closest equivalent to a Windows **`.exe`** file in Linux is simply an **executable file**, but Linux usually **doesn't rely on a file extension** like `.exe`.

### Windows

```text
program.exe
```

Windows commonly uses extensions to identify executable files:

* `.exe` → executable program
* `.bat` → batch script
* `.cmd` → command script
* `.msi` → installer

### Linux

A Linux executable might look like:

```text
program
```

For example:

```bash
ls
bash
nmap
python3
```

These are executable files, even though they have **no `.exe` extension**.

Linux determines whether a file can be executed mainly through:

1. **File permissions** — does it have the execute (`x`) permission?
2. **File format** — for compiled programs, commonly ELF.
3. **Shebang** — for scripts, such as `#!/bin/bash`.

### Example

Suppose you have:

```text
myprogram
```

Check its permissions:

```bash
ls -l myprogram
```

You might see:

```text
-rwxr-xr-x 1 user user 12345 myprogram
```

The `x` means it is executable.

You can run it using:

```bash
./myprogram
```

### Linux's equivalent of `.exe`: ELF

Most compiled Linux executables use the **ELF format**:

> **ELF = Executable and Linkable Format**

You can check a file with:

```bash
file /usr/bin/nmap
```

Example output:

```text
ELF 64-bit LSB pie executable
```

So roughly:

| Windows                                    | Linux                                       |
| ------------------------------------------ | ------------------------------------------- |
| `program.exe`                              | `program`                                   |
| PE (Portable Executable) format            | ELF format                                  |
| `.exe` extension is important              | Extension usually doesn't matter            |
| Run: `program.exe`                         | Run: `./program`                            |
| Executable identified largely by extension | Executable permission + file format/shebang |

### Scripts can also be executable

For example, a Bash script:

```bash
script.sh
```

Make it executable:

```bash
chmod +x script.sh
```

Then run:

```bash
./script.sh
```

So the important idea is:

**Windows:** “This ends in `.exe`, so it's an executable.”

**Linux:** “This file has execute permission and contains a valid executable format or script interpreter instruction, so it can be executed.”

If you want, I can next explain **what actually happens internally when you type `./program` in Linux**, including ELF, the kernel, the loader, and shared libraries.

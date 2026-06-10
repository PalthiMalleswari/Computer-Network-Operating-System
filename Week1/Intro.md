# Week 1 — Operating System Fundamentals
### A complete, self-paced syllabus for software engineers (1 hour/day × 7 days)

---

## How to use this guide

This is your full Week 1. It is deliberately detailed — read it like a textbook chapter, not a cheat sheet. Each day is built the same way so you fall into a rhythm:

1. **Concept** — what the thing is, in plain language.
2. **Why it matters to you** — the engineering payoff, so you know why you're learning it.
3. **Deeper reasoning** — the *why behind the why*, the part most tutorials skip.
4. **Visual model** — an ASCII diagram to anchor the idea spatially.
5. **Hands-on session** — exact commands, the output you should expect, and a line-by-line reading of that output.
6. **Reflection** — write 2–3 sentences in your own words. This is the step that makes it stick.

**Your daily 60 minutes, suggested split:**

| Minutes | Activity |
|---|---|
| 0–25 | Read the day's Concept, Why, and Reasoning sections |
| 25–55 | Run the hands-on session; actually type the commands, don't just read them |
| 55–60 | Write your reflection in your own words |

> **The single most important habit:** type every command yourself and read its output before moving on. Reading *about* `strace` teaches you nothing; running it on a real process rewires how you think about programs. The reflection writing at the end of each day is not optional — explaining a concept back is the best test of whether you truly understood it.

A realistic expectation: **Days 4 and 5 (memory layout and context switching) are the hardest.** If they take longer or need a second pass, that's normal and expected. Don't sacrifice understanding to stay "on schedule."

---

## Environment setup (do this before Day 1)

You need a real Linux environment. Pick your platform:

**Windows** — install WSL2 (Windows Subsystem for Linux). Open PowerShell *as Administrator* and run:
```powershell
wsl --install
```
Restart your machine when prompted. After restart, search for "Ubuntu" in your Start menu and open it. Create a username and password when asked. You now have a real Linux terminal.

**macOS** — open the **Terminal** app (Applications → Utilities → Terminal). macOS is Unix-based, so most commands work. A few low-level tools (`strace`, `/proc`) don't exist on macOS — where that happens, this guide tells you the macOS equivalent or says to just read along.

**Linux** — open your terminal. You're already set.

**Verify your setup** by running:
```bash
uname -a
```
If you see a line describing your system and kernel, you're ready. Example on Ubuntu/WSL:
```
Linux DESKTOP-AB12CD 5.15.167.4-microsoft-standard-WSL2 #1 SMP ... x86_64 GNU/Linux
```

---

## The big picture (read once, keep in your head all week)

Your computer is a stack of layers. Hardware at the bottom, your apps at the top, and the **operating system in the middle**. The rule that explains almost everything this week:

> **No application reaches the hardware without going through the operating system.**

```
   ┌───────────────────────────────────────────────────┐
   │  USER SPACE — your applications                     │
   │  browser · editor · games · terminal · your code    │
   └───────────────────────────────────────────────────┘
                  │  system calls (the ONLY doorway)  ▲
                  ▼                                    │
   ┌───────────────────────────────────────────────────┐
   │  KERNEL SPACE — the operating system                │
   │  scheduler · memory manager · file system · drivers │
   │  privileged mode, controls ALL hardware             │
   └───────────────────────────────────────────────────┘
                          │
                          ▼
   ┌───────────────────────────────────────────────────┐
   │  HARDWARE                                           │
   │  CPU · RAM · disk · network card                    │
   └───────────────────────────────────────────────────┘
```

The four big jobs of the OS are the spine of this week:

- **Day 1–2:** what the OS is, and the user/kernel boundary that makes it safe
- **Day 3–4:** processes, threads, and how a process is laid out in memory
- **Day 5:** how one CPU appears to run hundreds of programs at once
- **Day 6:** the Linux filesystem and permissions (daily practical skill)
- **Day 7:** tie it all together

Keep the layered picture above in mind. Every day this week is just zooming into one part of it.

---
---

# DAY 1 — What an operating system actually does

## Concept

An operating system has **one core job**: it is a **resource manager and a referee**.

Your machine has scarce resources — one CPU (or a handful of cores), a fixed amount of RAM, one disk — but dozens of programs that all want those resources *at the same time*. The OS decides who gets what, when, and for how long, while guaranteeing that no program can corrupt another's memory or crash the whole machine.

It carries out this job through four responsibilities:

1. **Process management** — loading and running your programs, and switching between them (Days 3, 5).
2. **Memory management** — giving each program its own private memory, safely (Day 4).
3. **File system management** — organizing data on disk into files and directories (Day 6).
4. **Device / I/O management** — talking to the keyboard, screen, disk, and network on programs' behalf.

## Why it matters to you

When your app is slow, crashes with "out of memory," hangs, or throws "permission denied," you are almost always colliding with one of those four responsibilities. Engineers who understand the OS turn these from mysterious, scary bugs into things they can reason about calmly. This single mental shift is worth the entire week.

## Deeper reasoning — the analogy that unlocks it

Think of the OS as the **manager of a busy restaurant kitchen**:

- There is **one stove** (the CPU), **limited counter space** (RAM), and a **pantry** (disk).
- Many **orders** (programs) arrive at once.
- The manager assigns cooks to stations, makes sure two cooks never grab the same pan, and keeps orders flowing so the kitchen *feels* like everything cooks simultaneously — even though the stove can only do so much at any single instant.

That last point is the OS's most important illusion: it makes a machine that can really only do one thing per core at a time *feel* like it's doing everything at once. You'll see exactly how that trick works on Day 5. Hold onto the kitchen image — we'll return to it.

## Visual model

```
        Many programs want resources          The OS decides
        ┌──────────────────────────┐          who gets the CPU,
        │ browser  music  download  │          memory, and disk —
        │ editor   chat   antivirus │   ───►   and enforces that
        └──────────────────────────┘          nobody breaks anybody
                                               else's stuff.
                  ▼  all funnel through the OS  ▼
        ┌──────────────────────────────────────────────┐
        │   OS = resource manager + referee              │
        │   • shares the CPU   • protects memory         │
        │   • manages files    • handles devices         │
        └──────────────────────────────────────────────┘
                  ▼
        ┌──────────────────────────────────────────────┐
        │   Hardware: 1 CPU (few cores), RAM, disk       │
        └──────────────────────────────────────────────┘
```

## Hands-on session

You're going to *meet* your OS. Run each command and read the explanation.

**1. Identify your OS and kernel:**
```bash
uname -a
```
Expected output (yours will differ in details):
```
Linux DESKTOP-AB12CD 5.15.167.4-microsoft-standard-WSL2 #1 SMP x86_64 GNU/Linux
```
Reading it field by field:
- `Linux` — the kernel name (the core of the OS).
- `DESKTOP-AB12CD` — your machine's hostname.
- `5.15.167.4-...` — the kernel **version**. This is the actual OS code running underneath everything.
- `x86_64` — your CPU architecture (64-bit Intel/AMD). On Apple Silicon Macs you'd see `arm64`.
- `GNU/Linux` — the broader OS environment.

**2. Who are you, to the OS?**
```bash
whoami
```
Output:
```
yourname
```
Every program you run, runs *as a user*. The OS uses your identity to decide what you're allowed to touch. This becomes central on Day 6 (permissions).

**3. How long has the OS been refereeing?**
```bash
uptime
```
Output:
```
 14:32:07 up 2 days,  3:11,  1 user,  load average: 0.18, 0.22, 0.25
```
Reading it:
- `up 2 days, 3:11` — the OS has been continuously managing resources for over two days.
- `1 user` — one logged-in user.
- `load average: 0.18, 0.22, 0.25` — average number of processes wanting the CPU over the last 1, 5, and 15 minutes. As a rule of thumb, if this number stays well below your core count, the machine isn't overloaded. (You'll understand exactly what "wanting the CPU" means after Day 5.)

**4. See how much memory the OS is managing:**
```bash
free -h
```
Output (`-h` = human-readable units):
```
               total        used        free      shared  buff/cache   available
Mem:            15Gi       3.2Gi       9.1Gi       180Mi       3.1Gi        11Gi
Swap:          4.0Gi          0B       4.0Gi
```
Reading it:
- `total` — total RAM the OS has to share among all programs.
- `used` — currently handed out to running programs.
- `available` — roughly how much a new program could get.
- `Swap` — disk space used as emergency overflow when RAM fills up (more on this in Week 11).

You've just watched the OS doing its resource-manager job: tracking memory, tracking time, tracking who you are.

## Reflection (write this down)

Complete this sentence in your own words: *"The operating system is the layer that ______, and its single most important job is ______."* Then list the four responsibilities from memory without looking back.

---
---

# DAY 2 — Kernel, user space, and the two CPU modes

## Concept

Yesterday's diagram had a line between "user space" and "kernel space." Today we make that line real, because it is one of the most important safety mechanisms in all of computing.

The **CPU itself** can run in two modes:

- **User mode** — *restricted.* Your applications run here. They cannot directly touch hardware, read another program's memory, or run privileged instructions. If they try, the CPU stops them.
- **Kernel mode** — *privileged.* The OS kernel runs here. It can do anything: access any memory, command any device.

The **kernel** is the core of the OS — the always-running program that lives in kernel mode and manages everything. Your browser is **not** part of the kernel; it is an ordinary user-mode program.

So how does a user-mode program (your browser) save a file to disk, if it cannot touch the disk? Through a **system call** — a controlled, checked request to the kernel.

## Why it matters to you

Two huge practical consequences flow from this design:

1. **A buggy app crashes itself, not your whole computer.** User mode contains the blast radius. This is why one frozen tab doesn't take down your entire OS.
2. **System calls cost time.** Each one triggers a switch from user mode into kernel mode and back. When you read advice like "reduce syscalls to make I/O faster," or you hear that "context switches are expensive," this boundary is the reason. Knowing this guides real performance decisions.

## Deeper reasoning — the bank vault analogy

Picture a bank:

- **You (user mode)** cannot walk into the **vault (hardware)**. That would be chaos — anyone could take anything.
- Instead you hand a **request slip (a system call)** to the **teller**.
- The teller **steps behind the secure counter (switches to kernel mode)**, performs the operation in the controlled area, and hands you back the result.
- The vault is *never* directly exposed to customers.

When your code runs `open("file.txt")`, it triggers an `open` system call. The CPU switches from user mode to kernel mode, the kernel **checks your permissions**, opens the file, and switches back to user mode with the result. On a busy machine this user↔kernel transition happens *millions of times per second*.

This is also *why* the permission checks on Day 6 are enforceable: every sensitive action passes through the kernel, which is the one place that can say "no."

## Visual model

```
   USER MODE (restricted)                KERNEL MODE (privileged)
   ┌─────────────────────┐               ┌──────────────────────────┐
   │  your program        │               │  the kernel               │
   │                      │  syscall      │                           │
   │  open("file.txt") ───┼──────────────►│  1. check permissions     │
   │                      │  (request)    │  2. talk to disk hardware  │
   │                      │               │  3. open the file          │
   │  ◄───────────────────┼───────────────┤  4. return a result        │
   │  use the result      │   return      │                           │
   └─────────────────────┘               └──────────────────────────┘
        cannot touch                          can touch
        hardware directly                     everything
```

The arrow crossing the boundary — twice — is the part that costs time.

## Hands-on session

You're going to *watch system calls happen in real time*. This is one of the most eye-opening exercises of the week.

> **Platform note:** `strace` is a Linux tool, available in WSL and Linux. On macOS the closest equivalent is `sudo dtruss <command>`, which is often blocked by security settings — if so, just read along; the concept is identical.

**1. Count the system calls a simple command makes:**
```bash
strace -c ls
```
This runs `ls` and prints a summary of every system call it made. Expected output (abbreviated):
```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 24.10    0.000142           7        19           mmap
 18.20    0.000107           8        13           openat
 14.50    0.000085           6        13           close
 11.30    0.000067           5        12           read
  ...
------ ----------- ----------- --------- --------- ----------------
100.00    0.000589                   95         2   total
```
Reading it: a command as trivial as `ls` made **95 system calls**. Every single one was your program crossing into kernel mode to ask for something it could not do alone — open files (`openat`), read them (`read`), map memory (`mmap`), close them (`close`). `ls` *looks* like one simple action to you, but underneath it is dozens of negotiated requests to the kernel.

**2. See the actual calls, not just the summary:**
```bash
strace ls 2>&1 | head -25
```
(`2>&1` redirects the trace output so we can pipe it; `head -25` shows the first 25 lines.) Expected output (abbreviated):
```
execve("/usr/bin/ls", ["ls"], 0x7ffe...) = 0
brk(NULL)                               = 0x55c8...
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0...", 832)      = 832
close(3)                                = 0
openat(AT_FDCWD, ".", O_RDONLY|O_...)   = 3
write(1, "Documents  Downloads  file.txt\n", 31) = 31
close(3)                                = 0
exit_group(0)                           = 0
```
Reading a few key lines:
- `execve("/usr/bin/ls", ...)` — the very first syscall: "kernel, please load and run this program."
- `openat(..., ".", ...) = 3` — "open the current directory." The `= 3` is a **file descriptor**, a small integer the kernel hands back to refer to that open file (you'll meet file descriptors again on Day 6).
- `write(1, "Documents Downloads...", 31)` — "write these 31 bytes to descriptor 1." Descriptor `1` is *standard output* — your terminal. **This is the line that actually puts text on your screen.** Your program cannot draw to the screen itself; it asks the kernel to.
- `exit_group(0)` — "I'm done, exit code 0 (success)."

Sit with this for a moment: the simple act of listing a folder is, underneath, a conversation of dozens of permission-checked requests across the user/kernel boundary. That conversation is the system-call interface.

## Reflection (write this down)

Answer in your own words: *Why can't your browser write to disk directly? What mechanism does it use instead, and why does that mechanism cost time?* Then explain what the `= 3` meant in the `openat` line.

---
---

# DAY 3 — Processes vs threads

## Concept

Precise definitions first, because these words get misused constantly:

- A **program** is *passive*: a file sitting on disk (e.g. `/usr/bin/firefox`). It does nothing until run.
- A **process** is a program that is *running*: the program loaded into memory, given resources, and executing. Launching the browser creates a process.

Each process gets its own:
- a private **memory space** (its own address space — Day 4),
- a list of **open files**,
- a unique **Process ID (PID)**,
- and **at least one thread**.

A **thread** is a single sequence of execution *inside* a process. Every process starts with one thread (the "main thread") and can create more.

The crucial distinction:

> **Threads within the same process share that process's memory. Separate processes do not share memory by default.**

## Why it matters to you

You will write multithreaded code constantly — web servers, background jobs, UIs that stay responsive while working. Shared memory between threads is simultaneously your most powerful tool and your most dangerous trap:

- **Powerful:** threads can hand each other data instantly, with no copying.
- **Dangerous:** if two threads modify the same data at the same moment, you get a **race condition** — a bug that appears randomly and is brutal to reproduce. (You'll study how to prevent these with locks and semaphores in Week 9.)

Understanding process-vs-thread also explains real architecture decisions, like why Chrome puts each tab in a **separate process**: so one tab crashing (its process dying) cannot corrupt or kill the others.

## Deeper reasoning — the house-and-residents analogy

This analogy is worth memorizing:

> **A process is a house. Threads are the people living in it.**

- Each **house (process)** has its own walls, its own street address, its own kitchen and furniture (its private memory).
- The **people inside (threads)** all share that same kitchen and furniture. Convenient — they can hand each other things directly. Also dangerous — two people reaching for the same knife at the same instant is a problem (the race condition).
- People in **different houses (different processes)** cannot just walk into each other's kitchens. To exchange something, they must send a letter (**inter-process communication** — pipes, sockets, shared-memory regions; covered in Week 7).

This is exactly why threads are "lightweight" (they reuse the house) and processes are "heavyweight" (each needs a whole new house built).

## Visual model

```
   PROCESS A (PID 101)                    PROCESS B (PID 102)
   ┌──────────────────────────────┐       ┌──────────────────────────────┐
   │  private memory (house A)      │       │  private memory (house B)      │
   │                                │       │                                │
   │  ┌──────────┐  ┌──────────┐    │       │  ┌──────────┐                  │
   │  │ Thread 1 │  │ Thread 2 │    │       │  │ Thread 1 │                  │
   │  │  (main)  │  │ (worker) │    │       │  │  (main)  │                  │
   │  └────┬─────┘  └────┬─────┘    │       │  └────┬─────┘                  │
   │       │  share      │          │       │       │                        │
   │       ▼             ▼          │       │       ▼                        │
   │  ┌──────────────────────────┐  │       │  ┌──────────────────────────┐  │
   │  │ shared heap + open files │  │       │  │ its own heap + files     │  │
   │  └──────────────────────────┘  │       │  └──────────────────────────┘  │
   └──────────────────────────────┘       └──────────────────────────────┘
        ▲                                          ▲
        └─ threads here CANNOT reach process B's memory, and vice versa ─┘
                 (to communicate, they must "send a letter": IPC)
```

## Hands-on session

**1. List running processes with their PIDs:**
```bash
ps aux | head -10
```
(`ps` = process status; `aux` = all processes, with user and detail; `head -10` = first 10 lines.) Expected output:
```
USER       PID %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root         1  0.0  0.1 168140 11400 ?     Ss   10:01   0:02 /sbin/init
root       412  0.0  0.2 240100 18900 ?     Ssl  10:01   0:01 /usr/lib/systemd
yourname  1820  2.3  4.1 998200 65000 pts/0 Sl   14:10   0:09 /usr/bin/firefox
yourname  2001  0.0  0.0  12100  3400 pts/0 R+   14:33   0:00 ps aux
```
Reading the columns:
- `USER` — who owns the process.
- `PID` — its unique Process ID. Note **PID 1** (`/sbin/init` or `systemd`): the very first process the OS starts at boot; every other process descends from it.
- `%CPU` / `%MEM` — share of CPU and memory it's using right now.
- `VSZ` / `RSS` — virtual memory size and actual RAM used (you'll fully understand these in Week 11).
- `STAT` — process state (`R` running, `S` sleeping, etc.).
- `COMMAND` — the program this process is running. Notice `ps aux` itself appears — listing processes is itself a process.

**2. See threads, not just processes:**
```bash
ps -eLf | head -10
```
The `-L` flag reveals threads. Expected output (abbreviated):
```
UID        PID  PPID   LWP  NLWP CMD
yourname  1820     1  1820    14 /usr/bin/firefox
yourname  1820     1  1825    14 /usr/bin/firefox
yourname  1820     1  1826    14 /usr/bin/firefox
```
Reading it:
- `PID` is the same (`1820`) on all three lines — they belong to the **same process** (the same house).
- `LWP` (Light-Weight Process = thread ID) is **different** on each line — these are **separate threads** (different residents).
- `NLWP` = `14` — this single Firefox process has **14 threads** all sharing its memory.

You're directly observing the house-and-residents model: one process, many threads inside it.

**3. Watch a process get created and destroyed:**
```bash
sleep 60 &
```
The `&` runs `sleep` in the background. Output:
```
[1] 24531
```
That `24531` is the PID of a brand-new process you just created. Confirm it exists:
```bash
ps aux | grep sleep
```
Output:
```
yourname  24531  0.0  0.0   8200   900 pts/0 S  14:40  0:00 sleep 60
```
There it is — a living process. After 60 seconds it finishes on its own and disappears. To kill it immediately instead:
```bash
kill 24531
```
You have now created and destroyed a process by hand — the most basic operation in process management.

## Reflection (write this down)

In your own words: *What's the difference between a program, a process, and a thread? What do threads share that separate processes do not, and why does that sharing create both an opportunity and a danger?* Use the house analogy.

---
---

# DAY 4 — How a process is laid out in memory

## Concept

When the OS creates a process, it hands that process a private **address space** — the process's own view of memory, as if it owned the entire machine. (It doesn't; the OS maintains this as an illusion for every process simultaneously. The full mechanism — *virtual memory* — is Week 11. For now, just accept that each process believes it has its own clean memory.)

That address space is organized into four main regions. Knowing them explains an enormous amount of real-world behavior:

- **Stack** — grows *downward* (from high addresses toward low). Holds **function call frames**: local variables, function arguments, and return addresses. Every function call *pushes* a frame; every return *pops* it. Fast, automatic, but limited in size.
- **Heap** — grows *upward*. Memory you request **explicitly at runtime** (`malloc` in C; `new` in many languages; object creation in higher-level runtimes). You — or your garbage collector — control when it's released.
- **Data** — global and static variables; things that exist for the whole program's life.
- **Text (code)** — the actual machine instructions of your program. Usually read-only, so a bug can't accidentally overwrite the code.

## Why it matters to you

Two of the most famous categories of bug map *directly* onto this layout:

- **Stack overflow** (yes, the website is named after this) — too many nested function calls fill the stack until it collides with the heap region. The overwhelmingly common cause is **infinite recursion** (a function that calls itself with no stopping condition). The process dies instantly.
- **Memory leak** — you keep allocating on the heap and never free it. The heap grows and grows until the process runs out of memory and is killed by the OS.

"Stack vs heap" is also one of the **most common technical interview questions**, and it explains concrete performance differences in your code: stack allocation is essentially free (just move a pointer), while heap allocation is slower and needs bookkeeping.

## Deeper reasoning — why two regions growing toward each other?

Why put the stack at the top growing down, and the heap at the bottom growing up? Because at compile time the OS *cannot know* how much of each a program will need. By placing them at opposite ends of the address space and growing them toward each other, the program gets maximum flexibility: a recursion-heavy program can have a huge stack and small heap, while an allocation-heavy program can have the reverse. They only fail if they actually *meet* in the middle — which is exactly what a stack overflow or out-of-memory condition is.

- **Stack discipline (LIFO — last in, first out):** when `main()` calls `foo()` which calls `bar()`, frames stack up as `[main][foo][bar]`. When `bar` returns, its frame pops; then `foo`; then `main`. This is automatic and is *why* local variables vanish when a function returns — their frame is gone.
- **Heap discipline (manual / GC):** heap memory outlives the function that created it. That's the whole point — you put data there when it needs to survive beyond a single function call. The cost is that *something* must remember to free it.

## Visual model

```
   HIGH ADDRESS
   ┌──────────────────────────────────────────┐
   │  STACK                                     │  ← function calls, local vars
   │  (locals, arguments, return addresses)     │
   │              │                             │
   │              ▼  grows DOWN                 │
   │                                            │
   │         (free space in between)            │
   │                                            │
   │              ▲  grows UP                   │
   │              │                             │
   │  HEAP                                      │  ← runtime allocations
   │  (malloc / new / objects)                  │     (you/GC free these)
   ├──────────────────────────────────────────┤
   │  DATA   (global + static variables)        │
   ├──────────────────────────────────────────┤
   │  TEXT   (your compiled code, read-only)    │
   └──────────────────────────────────────────┘
   LOW ADDRESS

   A stack overflow = STACK grows down so far it crashes into HEAP.
   A memory leak    = HEAP grows up forever because nothing frees it.
```

## Hands-on session

You're going to look at the **real memory map of a real process** — the diagram above, made concrete.

> **Platform note:** `/proc` is a Linux feature (WSL and Linux). It does not exist on macOS. macOS users: read along, the layout concept is universal across operating systems.

**1. Start a long-lived process and get its PID:**
```bash
sleep 300 &
```
Output:
```
[1] 26104
```
Note the PID (here `26104`).

**2. View its actual memory regions:**
```bash
cat /proc/26104/maps | head -20
```
(`/proc/<PID>/maps` is a file the *kernel generates on the fly* describing that process's memory layout.) Expected output (abbreviated):
```
55a3c1e00000-55a3c1e02000 r-xp 00000000 08:01 1180   /usr/bin/sleep      ← text (code), r-x = read+execute
55a3c1e02000-55a3c1e03000 r--p 00002000 08:01 1180   /usr/bin/sleep      ← read-only data
55a3c1e03000-55a3c1e04000 rw-p 00003000 08:01 1180   /usr/bin/sleep      ← data (read+write)
55a3c3f50000-55a3c3f71000 rw-p 00000000 00:00 0      [heap]              ← THE HEAP
7f2e8c000000-7f2e8c021000 r-xp 00000000 08:01 2210   /lib/x86_64.../libc.so
7ffd5a3c0000-7ffd5a3e1000 rw-p 00000000 00:00 0      [stack]             ← THE STACK
```
Reading it:
- The lines labeled `/usr/bin/sleep` with permission `r-xp` are the **text segment** — the program's code, marked read + execute but *not* write (so it can't be overwritten).
- `[heap]` — literally the heap region from the diagram, marked `rw-` (read + write).
- `[stack]` — literally the stack region, also `rw-`.
- The `libc.so` line is the C standard library, loaded into the process's space.

You are looking at the exact four-region layout from the visual model, for a process running on your machine right now. The OS isn't hiding this from you — it's showing it to you through `/proc`.

**3. Clean up:**
```bash
kill 26104
```

**Optional — see a stack overflow happen (Python, safe to try):**
```bash
python3 -c "
import sys
def recurse(n):
    return recurse(n+1)   # calls itself forever, never returns
recurse(0)
"
```
Output (abbreviated):
```
RecursionError: maximum recursion depth exceeded
```
That error *is* a controlled stack overflow: each `recurse` call pushed a new frame onto the stack until the limit was hit. Python catches it gracefully; in a language like C, the same pattern crashes the process outright. You've now seen the stack region's limit in action.

## Reflection (write this down)

Name the four memory regions and what each holds. Then explain, using the regions: *what physically happens during (a) a stack overflow and (b) a memory leak?*

---
---

# DAY 5 — The illusion of "everything at once": context switching

## Concept

Here is the magic trick promised on Day 1. You might have 8 CPU cores but **300 running processes**. How does music play while you type while a download runs while antivirus scans?

The answer: the CPU **switches between processes incredibly fast** — giving each one a tiny slice of time (typically a few milliseconds), then moving to the next. It happens so quickly that, to a human, it *feels* perfectly simultaneous. This is **concurrency** achieved through **time-slicing**.

When the CPU switches from process A to process B, it must:

1. **Save** everything about A's current state — exactly where it was in its code, the values in its CPU registers — into A's record.
2. **Load** B's previously saved state.
3. **Resume** B exactly where it had left off, as if it were never interrupted.

This save-and-restore operation is called a **context switch**. The OS component that decides *who runs next* is the **scheduler** (you'll study scheduling algorithms — round robin, priority, and more — in detail in Week 5).

## Why it matters to you

This is the bedrock of everything concurrent you will ever build or debug:

- It's *why* a single server can "handle 10,000 connections."
- It explains a counterintuitive trap: spawning **5,000 threads** to "go faster" often makes things **slower**, because the CPU spends all its time context-switching (saving and restoring state) instead of doing real work. Understanding this stops you from making a classic performance mistake.
- It explains why "the system feels sluggish under heavy load" — too many things competing means more time lost to switching overhead.

## Deeper reasoning — the chess grandmaster analogy

> A **chess grandmaster** plays **20 opponents at once**, walking down a long row of boards.

- At each board, the grandmaster **restores the game state** in their head (loads that game's "context"), **makes one move**, then **walks to the next board**.
- From any single opponent's perspective, their game is *never paused* — it keeps progressing. The grandmaster gives the *illusion* of playing all 20 games simultaneously, despite having only one brain.
- **But the walking between boards is pure overhead.** If there were 5,000 boards, the grandmaster would spend the entire session walking and have no time left to actually think about moves.

That "walking" is the context switch. A little is fine and creates the magic of multitasking. Too much, and the system grinds — all walking, no thinking. This is *exactly* why more threads is not always faster.

## Visual model

```
   ONE CPU CORE, sliced across time  ──────────────────────────────────►

   │ A │ B │ C │ A │ B │ C │ A │ B │ C │ ...
   └─┬─┴─┬─┴─┬─┴───┴───┴───┴───┴───┴───┘
     │   │   │
     │   │   └─ each switch: SAVE current process's state,
     │   │                   LOAD next process's state  = a CONTEXT SWITCH
     │   └───── process B runs for ~a few milliseconds
     └───────── process A runs for ~a few milliseconds

   To you it looks like A, B, and C all run at the same time.
   In reality the core does a few ms of each, switching ~constantly.

   The chess grandmaster:
       board A ─► move ─► [walk] ─► board B ─► move ─► [walk] ─► board C ...
                          ▲ the "walk" = context switch = overhead
```

## Hands-on session

You're going to *watch context switches happen* as a live number.

**1. Watch system activity, including context switches:**
```bash
vmstat 1 5
```
(`vmstat` = virtual memory & system stats; `1` = update every 1 second; `5` = print 5 times.) Expected output:
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 9320000  18000 310000    0    0     5    12  210  480  2  1 97  0  0
 0  0      0 9319000  18000 310000    0    0     0     0  198  455  1  1 98  0  0
 1  0      0 9319500  18000 310000    0    0     0     8  205  470  2  1 97  0  0
```
Reading the key columns:
- **`cs`** — **context switches per second.** This is the star of today's lesson. Even an idle machine does hundreds per second — the OS is *always* juggling.
- `in` — hardware interrupts per second.
- `r` — processes currently *runnable* (waiting for or using the CPU). This is what "load average" from Day 1 was measuring.
- `us` / `sy` / `id` — percent of CPU time in user mode / kernel (system) mode / idle. Notice `id` is high (97–98) on an idle machine — the CPU is mostly waiting for work.

**2. Create load and watch `cs` jump:**

In one terminal, start a busy process:
```bash
yes > /dev/null &
```
(`yes` floods output as fast as it can; `> /dev/null` throws it away; `&` backgrounds it. This deliberately keeps the CPU busy.) Now run `vmstat` again:
```bash
vmstat 1 5
```
You'll see the `cs` and `us` (user CPU) numbers climb noticeably compared to before — the system is now busier, and the scheduler is working harder. **Kill the busy process when done:**
```bash
kill %1
```
(`%1` refers to background job 1.)

**3. See per-core activity live:**
```bash
top
```
Once `top` is open, **press `1`** — it expands to show each CPU core separately. Press **`q`** to quit. Each core is being time-sliced among many processes; `top` shows you the result of all that switching as live percentages.

You've now watched, as a real number, the mechanism that creates the illusion of a computer doing everything at once.

## Reflection (write this down)

Explain in your own words: *How does one CPU core appear to run hundreds of programs simultaneously? What exactly is saved and restored during a context switch, and why does doing it too often hurt performance?* Use the grandmaster analogy.

---
---

# DAY 6 — The Linux filesystem and permissions

## Concept

Today shifts from theory to a **practical skill you'll use every single day** as an engineer.

**Part A — "Everything is a file."** In Linux, nearly everything is represented as a file in a single tree that starts at the root, `/`. You already saw this yesterday: `/proc/<PID>/maps` *looked* like a file but was really live kernel info. Key directories:

| Directory | What lives there |
|---|---|
| `/home` | User files (your personal stuff) |
| `/etc` | System-wide configuration files |
| `/bin`, `/usr/bin` | Programs / commands (like `ls`, `cat`) |
| `/var` | Variable data — logs, caches |
| `/tmp` | Temporary files (often cleared on reboot) |
| `/proc` | Live system info, generated by the kernel (not real disk files) |

**Part B — Permissions.** Every file has an **owner**, a **group**, and three permission types — **read (r)**, **write (w)**, **execute (x)** — defined separately for three categories of people: **owner**, **group**, and **others** (everyone else).

When you run `ls -l`, the leftmost field, like `-rwxr-xr--`, encodes all of this:

```
   -  rwx  r-x  r--
   │   │    │    │
   │   │    │    └── others:  r--  (read only)
   │   │    └─────── group:   r-x  (read + execute)
   │   └──────────── owner:   rwx  (read + write + execute)
   └──────────────── type:    -    (regular file; 'd'=directory, 'l'=link)
```

Each permission also has a **number**: read = **4**, write = **2**, execute = **1**. Add them up per category:

- `rwx` = 4+2+1 = **7**
- `r-x` = 4+0+1 = **5**
- `r--` = 4+0+0 = **4**

So `-rwxr-xr--` is **754** in numeric (octal) form. This is why you'll constantly see commands like `chmod 755 script.sh` — it's setting those exact permission bits.

## Why it matters to you

This is not abstract. In your first week at any engineering job you *will* hit this: you write a deploy script, try to run it, and get:
```
bash: ./deploy.sh: Permission denied
```
The fix is almost always:
```bash
chmod +x deploy.sh
```
— adding the **execute** bit. It happens to everyone, and now you'll know *exactly* why: the file existed and was readable, but without the `x` bit the kernel refused to run it. (And recall from Day 2 *why* the kernel can enforce this: every attempt to execute passes through kernel mode, the one place that can check permissions and say no.)

You'll also use permissions to reason about security ("why can this service read that secret file?"), debugging, and deployment constantly.

## Deeper reasoning — why three categories and a numeric form?

The owner/group/others split exists so the system can serve many users at once safely (recall the OS is a *referee*, Day 1). Your files can be private to you (owner), shared with a team (group), or public (others) — all without separate copies. The numeric/octal form exists because computers represent these three bits (r, w, x) as a single digit 0–7 internally; `chmod 754` is simply you speaking the kernel's native language for permissions. Once you see that 7 = 111 in binary = rwx all on, and 5 = 101 = r-x, the numbers stop being arbitrary.

## Visual model

```
   THE FILESYSTEM TREE                 A PERMISSION STRING DECODED
        /  (root)                        - rwx r-x r--
        ├── home/                        │  │   │   └ others:  read
        │   └── yourname/  ← your stuff  │  │   └──── group:   read+exec
        ├── etc/    ← config             │  └──────── owner:   read+write+exec
        ├── bin/    ← commands           └─────────── type: regular file
        ├── usr/
        │   └── bin/
        ├── var/    ← logs                 numeric form:
        ├── tmp/    ← temp                   rwx = 7   r-x = 5   r-- = 4
        └── proc/   ← live kernel info       →  this file is  754

                                          r=4   w=2   x=1   (add per group)
```

## Hands-on session

**1. Read real permission strings:**
```bash
ls -l
```
Expected output:
```
total 12
drwxr-xr-x 2 yourname yourname 4096 Jun 10 14:00 projects
-rw-r--r-- 1 yourname yourname  220 Jun 10 13:55 notes.txt
-rwxr-xr-x 1 yourname yourname  812 Jun 10 13:58 run.sh
```
Reading line 1 (`projects`):
- `d` — it's a **directory**.
- `rwxr-xr-x` — owner can read/write/enter; group and others can read/enter but not write. (For directories, `x` means "can enter/traverse.")
- `yourname yourname` — owner and group.
- `4096` — size in bytes; `Jun 10 14:00` — last modified.

Reading line 3 (`run.sh`): `-rwxr-xr-x` → owner has the execute bit (`x`), so this script can be run by its owner. Numeric form: `755`.

**2. Look at system config permissions:**
```bash
ls -l /etc | head
```
You'll see system files, many owned by `root`, often `-rw-r--r--` (644: owner can edit, everyone else read-only). This is the OS protecting critical configuration from accidental edits by normal users.

**3. The "permission denied" experience — and the fix:**
```bash
echo 'echo "Hello from my script!"' > test.sh   # create a simple script
./test.sh                                        # try to run it
```
Output:
```
bash: ./test.sh: Permission denied
```
Check why with `ls -l test.sh`:
```
-rw-r--r-- 1 yourname yourname 33 Jun 10 14:45 test.sh
```
Notice: **no `x` anywhere.** The file is readable but not executable, so the kernel refuses to run it. Now fix it:
```bash
chmod +x test.sh
./test.sh
```
Output:
```
Hello from my script!
```
Confirm what changed:
```bash
ls -l test.sh
```
Output:
```
-rwxr-xr-x 1 yourname yourname 33 Jun 10 14:45 test.sh
```
The `x` bits appeared. You just diagnosed and fixed the single most common permission issue in software engineering.

**4. Practice the numeric form — predict before you run:**
```bash
chmod 644 test.sh
ls -l test.sh
```
Before running `ls`, predict the string. `6`=rw-, `4`=r--, `4`=r-- → you should expect `-rw-r--r--`. Run it and confirm you were right. Try a few more: `chmod 700 test.sh` (predict `-rwx------`), `chmod 755 test.sh` (predict `-rwxr-xr-x`). When your predictions match reliably, you've internalized permissions.

## Reflection (write this down)

Decode `-rwxr-x---` for all three categories and give its numeric form. Then explain in one sentence *why* `chmod +x` fixes a "permission denied" error, connecting it back to the kernel's role from Day 2.

---
---

# DAY 7 — Consolidation: tie it all together

## Concept

No new material today. This is the day that converts a week of reading into durable knowledge. **Spaced review is the difference between remembering this in six months versus forgetting it in six days.** Treat today as seriously as any other.

## Self-quiz (write answers *before* checking back)

Without looking at the earlier days, answer each in your own words on paper or in a notes file:

1. What is the single core job of an operating system, and what are its four responsibilities?
2. Why can't your browser write to disk directly? What mechanism does it use, and why does that mechanism cost time?
3. What is the difference between a program, a process, and a thread? What do threads share that separate processes do not?
4. Name the four regions of a process's memory and what each holds. What happens, region-wise, in a stack overflow and in a memory leak?
5. How does one CPU core appear to run hundreds of programs at once? What is saved/restored in a context switch, and why does doing it too much hurt performance?
6. Decode the permission string `-rwxr-xr--`: what can each category do, and what is the numeric form? Why does `chmod +x` fix "permission denied"?

After writing your answers, check them against the relevant day. **Any question you couldn't answer confidently → reread that day and redo its hands-on session.** That targeted review is the highest-value 20 minutes of your week.

## Capstone exercise — connect Days 3, 4, and 5 in one flow

This single session exercises process creation, memory layout, and context switching together.

```bash
# --- Day 3: create a process and find it ---
sleep 500 &
```
Output: `[1] 28470` (note your PID).
```bash
ps aux | grep sleep
```
Confirm your living process appears, with its PID, owner, and state.

```bash
# --- Day 4: inspect THIS process's memory layout ---
cat /proc/28470/maps | grep -E "stack|heap"
```
Output (abbreviated):
```
55b8...-55b8... rw-p ... [heap]
7ffe...-7ffe... rw-p ... [stack]
```
You're seeing the real stack and heap regions of the process you just created.

```bash
# --- Day 5: watch context switches while it (and everything) runs ---
vmstat 1 3
```
Watch the `cs` column — context switches per second — proving the OS is constantly juggling, even with your `sleep` mostly idle.

```bash
# --- Clean up ---
kill 28470
```

**Final reflection — the test of mastery:** In 3–4 sentences, describe what you just did, correctly using the words **process**, **PID**, **stack**, **heap**, and **context switch**. For example: *"I created a new **process** with `sleep`, which the OS gave a unique **PID**. I inspected its memory and saw its **stack** and **heap** regions in `/proc`. While it ran, `vmstat` showed the **context switch** count, proving the CPU was rapidly switching between this and other processes to fake simultaneous execution."* If you can write that confidently from your own understanding, **Week 1 is solid** and you're ready for Week 2.

---
---

# Reference — Command cheat sheet

Keep this handy; you'll use these all the way through the course.

| Command | What it does | Introduced |
|---|---|---|
| `uname -a` | Show OS, kernel version, architecture | Day 1 |
| `whoami` | Show your current user | Day 1 |
| `uptime` | Uptime + load average | Day 1 |
| `free -h` | Memory usage, human-readable | Day 1 |
| `strace -c <cmd>` | Count system calls a command makes | Day 2 |
| `strace <cmd>` | Trace each system call in order | Day 2 |
| `ps aux` | List all processes with detail | Day 3 |
| `ps -eLf` | List processes *and their threads* | Day 3 |
| `sleep N &` | Start a background process for N seconds | Day 3 |
| `kill <PID>` | Terminate a process by PID | Day 3 |
| `kill %1` | Terminate background job 1 | Day 5 |
| `cat /proc/<PID>/maps` | Show a process's memory regions | Day 4 |
| `vmstat 1 5` | System stats incl. context switches (`cs`) | Day 5 |
| `top` (then `1`) | Live per-core CPU activity | Day 5 |
| `ls -l` | Long listing with permissions | Day 6 |
| `chmod +x <file>` | Add the execute permission | Day 6 |
| `chmod 755 <file>` | Set permissions numerically | Day 6 |

---

# Reference — Glossary

- **Operating system (OS)** — the software layer that manages a computer's resources and acts as referee between programs and hardware.
- **Kernel** — the core of the OS; the always-running program that operates in privileged (kernel) mode and controls all hardware.
- **User mode / kernel mode** — the two CPU privilege levels; apps run restricted in user mode, the kernel runs unrestricted in kernel mode.
- **System call** — a controlled, permission-checked request from a user-mode program asking the kernel to perform a privileged operation (e.g. open a file, write to the screen).
- **Program** — a passive file on disk containing instructions.
- **Process** — a running program: loaded into memory, given resources, and assigned a unique PID.
- **PID** — Process ID; a unique number identifying a process.
- **Thread** — a single sequence of execution inside a process; threads in one process share that process's memory.
- **Address space** — the private view of memory the OS gives each process.
- **Stack** — memory region for function call frames (locals, arguments, return addresses); grows down; fast and automatic.
- **Heap** — memory region for runtime allocations; grows up; freed manually or by a garbage collector.
- **Stack overflow** — crash caused by the stack growing too large, usually from infinite recursion.
- **Memory leak** — heap memory that is allocated but never freed, causing memory to grow until exhaustion.
- **Concurrency** — multiple tasks making progress over the same period, achieved on a single core via time-slicing.
- **Time-slicing** — giving each process a small slice of CPU time in turn.
- **Context switch** — saving one process's CPU state and loading another's so the CPU can switch between them.
- **Scheduler** — the OS component that decides which process runs next (studied in depth in Week 5).
- **Permissions (rwx)** — read/write/execute rights, defined per owner/group/others; expressible numerically (r=4, w=2, x=1).

---

*End of Week 1. When your self-quiz answers are confident and your capstone reflection is solid, you're ready to move on. Next up in your roadmap: Week 2 — Networking Foundations (the OSI and TCP/IP models).*

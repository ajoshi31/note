---
title: Interrupt and Exception Handling
toc: true
date: 2017-07-30
tags: [OS]
top: 7
---

## Our Simple Shell

```c
/* Disclaimer: this is C-like pseudo-code.
 It will not compile or run! (But it’s not far off.) */

while (1) {
  input = readLine();
  returnCode = fork();
  if (returnCode == 0) {
    exec(input);
  }
}
```

## `exec()` Challenges
The most challenging part of `exec()` is making sure that, on failure, `exec()` can return to the calling process!

* Can’t make destructive changes to the parent’s address space until we are sure that things will success.
* Of course, the process is just an abstraction anyway and that provides a lot of flexibility: can prepare a separate address space and just swap it in when we’re done.

## `exit()` # End of Life Issues

* What’s missing here? **Death**!
* Processes choose the moment of their own end by calling `exit()`.
* As we discussed earlier a processes passes an **exit code** to the `exit()` function.
* What happens to this exit code?

## `wait()` # The Afterlife

* When a process calls `exit()` the kernel holds the exit code, which can be retrieved by the exiting child’s parent.
* The parent retrieves this exit code by calling `wait()`, the last of the primary process-related system calls.
    * And the one that stubbornly refuses to fit into my lifecycle metaphor.

## wait()/exit()

* We often consider `wait()` and `exit()` together, since they combine to remove any trace of a process from the system.
* Until a process *both* calls `exit()` and has its exit code collected via `wait()` traces of it remain on the system:
    * Its return code is retained by the kernel.
    * Its process ID (or PID) is also retained. Why?
* Processes that have `exit()`ed but not had their exit code collected are called **zombies**. (Ooh, scary!)
* `wait()`/`exit()` also present an interesting synchronization problem you will solve for ASST2.
    * Calls to `wait()` (by the parent) and `exit()` (by the child) may interleave in the kernel.
    * You must guarantee that the parent can retrieve the exit code successfully.

## `wait()`/`exit()` Issues

* What happens if a process’s parent exits before it does?
    * The "orphaned" process is assigned the init process as a parent, which  will collect its exit code when it exits. Referred to as reparenting.
* How do we prevent zombies from taking over the machine?
    * A processes parent receives the SIGCHLD signal when a child calls exit(), alerting it to the chance to retrieve the child’s exit status.
    * On some systems a process can choose to have its children automatically reaped by ignoring this signal.
    * On bash the relevant command is the appropriately-named disown. This allows children to continue running as daemons even after bash exits.

## What If I Don’t Want to `wait()`?
* Parent may want to peek at the exit status of its child, just to check on it. (Are you dead yet? Are you dead yet?)
* Systems support a non-blocking `wait()` for this purpose:
    * **Blocking** `wait()` will block until the child exits, unless it has already exited in which case it returns immediately.
    * **Non-Blocking** `wait()` will not block. Instead, its return status indicates if the child has exited and, if so, what the exit code was.

## Our Simple Shell
* Disclaimer: this is C-like pseudo-code. It will not compile or run! (But it’s not far off.)

```c
while (1) {
  input = readLine();
  returnCode = fork();
  if (returnCode == 0) {
    exec(input);
  } else {
    wait(returnCode);
  }
}
```

## Aside: errno

Where’s `exit()`?
* There is potential confusion between kernel system calls and wrappers implemented by `libc`:
    * `_exit()` (system call) v. `exit()` (C library function call).
    * The C library wraps system calls and changes their return codes.
    * The C library is what sets errno, not the kernel.

## Multiplexing and Abstracting the CPU

For the next several weeks we’ll be looking at how the operating system manages the processor:
* What are the **limitations or problems** with the hardware resource that the operating system is trying to address? *There is only one (or at least, no that many) processor(s)*!
* What are the **mechanisms** necessary to allow the processor to be shared? *Interrupts and context switching*.
* What are the **consequences** for programmers of processor multiplexing? *Concurrency and synchronization*.
* How do we design good **policies** ensuring that processor sharing meets the needs of the user? *Processor scheduling*.

## Today: Operating System Privilege

* Earlier we alluded to the fact that the operating system is like a normal program with some special privileges.
* In fact, implementing most of the process-related system calls we discussed last week *does not require these special privileges*!
    * If you don’t believe me, look at user-space threading libraries. They provide functionality very similar to the `fork()`, `exec()`, `exit()` and `wait()` system calls we discussed.
* So *why does the operating system need special privileges*?

## Multiplexing Requires Privilege

* In many cases implementing abstractions does not require special privileges.
* However, the operating systems other task—multiplexing resources— **does**.
* In order to divide resources between processes the system needs a trusted and privileged entity that can:
    * **divide** the resources, and
    * **enforce** the division.

## No Trusto Processo

* Why can’t processes share resources without a privileged arbiter?
* Some processes are:
    * malicious—"Hey, I’d like some more memory, so I’ll use yours!"
    * buggy—"Um, is this my memory or your memory? I’m not sure but I’ll just use it and hope things turn out OK…​"

## Privileged Execution
* CPUs implement a mechanism allowing the operating system to manage resources: **kernel** (or privileged) **mode**.
* Being in kernel mode may mean that the executing code
    * has access to **special instructions**, and
    * has a different **view of memory**.

## Special Instructions

* When the CPU is in **kernel mode** there are special instructions that can be executed.
    * These instructions usually modify important global state controlling how resources are shared.
* When the CPU is not in kernel mode it does not allow these instructions to be executed.
    * We will see what happens when an unprivileged process tries to execute a privileged instruction in a minute.

## Protection Boundaries

* The goal:
    * only trusted kernel code runs in kernel mode;
    * untrusted user code always runs in user mode.
* The CPU implements mechanisms to transition between user and kernel mode which we will discuss during the rest of today’s class.

## Aside: Fine-Grained Protection

* Many modern CPUs implement **more than two** protection modes.
* x86 processors actually have four protection "rings" from Ring 0 (most privileged) to Ring 3 (least privileged).
* For many years operating systems running on x86 architectures only used Ring 0 (kernel mode) and Ring 3 (user mode).
* Recently this has become more interesting because of operating system virtualization, so we will return to this.
    * But for now, you can think of processors as having two privilege modes: kernel mode and user mode.

## Terminology

* When we say "*application*" we refer to code running without privileges or in unprivileged or "user" mode.

* When we say "*kernel*" we mean code running in privileged or kernel mode.

* What makes the kernel special? **It is the one application allowed to executed code in kernel mode!**

## Bootstrapping Privilege

Why is the operating system allowed to run in kernel mode?
* **You installed your machine that way**! This is what it means to install an operating system: choose a particular application to grant special privileges to.
* On boot the CPU starts out executing the kernel code in privileged mode, which is how **privilege is bootstrapped**.
* The kernel is responsible for lowering the privilege level before executing user code.

## More Terminology: Traps

* When a normal application does something that causes the system to enter **kernel mode** we sometimes refer to this as trapping	into the kernel.
* I frequently think about the thread that trapped into the kernel as **running in the kernel** after the trap occurs.
    * On some level this is accurate: it is the same stream of instructions.
    * On some level this is not accurate: the kernel thread has its own stack and has saved the state of the trapping user thread, so in a way the user thread has been paused while the kernel performs some task on its behalf.
* Decide the way to think about this that is the most effective for you.

## Privilege Transitions
* The transition into the kernel or into privileged mode typically occurs for one of three reasons:
    * a hardware device requests attention—**hardware interrupt**
    * software requests attention—**software interrupt** or system call
    * software needs attention—**software exception**
* What is the difference between *requesting* and *needing* attention?

## Hardware Interrupts

* Hardware interrupts are used to signal that a particular device needs attention:
    * a disk read completed, or
    * a network packet was received, or
    * a timer fired.
* Processors implement multiple *interrupt lines*, input wires on which a logic transition (or level) will trigger an interrupt.

## Interrupt Handling
When an interrupt is triggered (interrupt request, or IRQ), the processor:

1. **enters privileged mode**,
2. **records state** necessary to process the interrupt,
3. **jumps to a pre-determined memory location** and begins executing instructions.

The instructions that the processor executes when an interrupt fires are called the **interrupt service routine** (ISR).
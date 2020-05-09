---
title: Introduction to scheduling
toc: true
date: 2017-07-30
tags: [OS]
top: 10
---

## Thread States
We talk about threads—and sometimes the processes containing them—as being in several different states:

* **Running**: executing instructions on a CPU core.
* **Ready**: not executing instructions but capable of being restarted.
* **Waiting, Blocked or Sleeping**: not executing instructions and not able to be restarted until some event occurs.

## Thread State Transitions

* Running → Ready: a thread was **descheduled**.
* Running → Waiting: a thread performed a **blocking system call**.
* Waiting → Ready: the event the thread was waiting for happened.
* Ready → Running: a thread was **scheduled**.
* Running → Terminated: a thread **exited** or hit a fatal **exception**.

Operating systems have data structures to organize threads into these groups which you encountered during ASST1.

## Scheduling: What?
What is scheduling?

* **Scheduling** is the process of choosing the next thread (or threads) to run on the CPU (or CPUs).
* We will primarily discuss **single core** scheduling for most of the week but return to multi-core scheduling issues later.

## Scheduling: Why?
Why schedule threads?

* **CPU multiplexing**: we have more threads that cores to run them on.
* **Kernel privilege**: we are in charge of allocating the CPU and must try to make good decisions. **Applications rely on it.**

## Scheduling: When?
When does scheduling happen?

* When a thread **voluntarily** gives up the CPU by calling `yield()`.
* When a thread makes a blocking **system call** and must sleep until the call completes.
* When a thread **exits**.
* When the **kernel decides** that a thread has run for long enough.
* `#4` is what makes a scheduling policy **preemptive**, as opposed to **cooperative**: the kernel can preempt (or stop) a thread that has not requested to be stopped.

## Why `yield()`?

What is the rationale behind having a way for threads to voluntarily give up the CPU?

* `yield()` can be a useful way of allowing a well-behaved thread to tell the CPU that it has no more useful work to do.
* `yield()` is inherently cooperative. "Let me get out of the way so that another, more useful, thread can run."

## Scheduling: How?
Two separate questions here:

* **Mechanism**: how do we switch between threads?
* **Policy**: how do we choose the next thread to run?

How do we **switch between threads**?

* Perform a **context switch** and move threads between the **ready, running,** and **waiting** queues.

## Policy v. Mechanism
Scheduling is as example of useful separation between **policy** and **mechanism**:

* P: deciding what thread to run.
* M: context switch.
* M: maintaining the running, ready and waiting queues.
* P: giving preference to interactive tasks.
* M: using timer interrupts to stop running threads.
* P: choosing a thread to run at random.

## Scheduling Matters
How the CPU is scheduled impacts **every other part** of the system.

Why?

* Using other system resources requires the CPU!
* **Intelligent scheduling** makes a modestly-powered system seem fast and responsive.
* **Stupid scheduling** makes a powerful system seem sluggish and laggy.

## Human-Computer Interaction (and Expectations)
What do you expect from your machine?

* **Respond** (Click)
* **Continue** (Watch, or active waiting)
* **Finish** (Expect, or passive waiting)

## Respond (Click)
**Responsiveness**: when you give the computer and instruction, or input, **it responds in a timely manne**r.

* It may **not** finish, but at least you know it has **started** (or understood).

Most of what we do with computers consists of responsive tasks. This is **using** a computer, and what makes computers different from television.

Examples of responsive tasks:

* Web browsing: when a link is clicked, retrieve the web page.
* Editing: when I enter text at the keyboard, place it at the cursor.
* Chatting: when I hit send, transmit the text to my chat partner.

## Continue (Watch)
**Continuity**: when you ask the computer to perform a continuous task it does so smoothly.

* Continuity implies **active waiting**: you are not interacting with the computer, but you are expecting it to continue to perform a task you have initiated.

As computers have started to deliver media, this function is **increasingly important**.

Examples of continuous tasks:

* Blinking a cursor.
* Playing music or a movie.
* Stupid (!) web animations.

## Finish (Expect)
**Completion**: when we ask to the computer to perform a task—or it performs one on our behalf—that we expect to take a long time, we want it to complete eventually.

* Completion implies **passive waiting**: you are asking the computer to continue to deliver interactive performance while working on your long-running task. (We also consider these background tasks.)
* Unlike responsive and continuous task, background tasks may not be user initiated.

Examples of background tasks:

* Performing a system backup.
* Indexing files on my computer.

## Click, Watch, Expect
Many applications **combine** all three system expectations.

Music player:

* Click: change tracks.
* Watch: play the selected track.
* Finish: update album artwork.

Web browser:

* Click: follow a link.
* Watch: play web video.
* Finish: index search history.

## Conflicting Goals
Scheduling is a balance between **meeting deadlines** and **optimizing resource allocation**:

* Optimal resource allocation: carefully allocate tasks so that all resources are constantly in used.
* Meeting deadlines: drop everything and do this **now**!

Responsiveness and continuity require meeting deadlines—unpredictable or predictable:

* **Responsiveness** → unpredictable deadlines. "When the user moves the mouse I need to be ready to redraw the cursor."
* **Continuity** → predictable deadlines. "Every 5 ms I need to write more data to the sound card buffer."

Throughput requires careful resource allocation:

* **Throughput** → optimal resource allocation. "I should really give the backup process more resources so that it can finish overnight."

## Deadlines Win
Humans are sensitive to **responsiveness** and **continuity**.

* We don’t notice resource allocation (as much).
* Heard: "My computer feels slow."
* Unheard: "My computer is not using all of its RAM."

**Why**:

* Poor responsiveness or continuity wastes our time! ("The mouse jumped all over and I couldn’t click anywhere.", "The movie kept stalling and I couldn’t watch it.")
* Poor throughput usually just wastes computer time. ("The backup took 12 hours but I was sleeping.")

## Scheduling Goals
(Or, how to **evaluate schedulers**.)

* How well does it meet deadlines—unpredictable or predictable?
* How completely does it allocate system resources?
* No point having idle CPU, memory, or disk bandwidth when something useful could be happening.

On human-facing systems, deadlines (or **interactivity**) usually wins. Why?
* Your time is more valuable than your computer’s.

## (Aside) Realtime Scheduling

We have established that deadlines are important to human-facing systems. This is mainly because systems that don’t meet deadlines are **annoying**. ("Buffering…​", "Buffering…​", etc.)

There are other classes of systems where the failure to meet deadlines can be **fatal**.

* "I meant to get around to running the motion_stop task 1 s ago, but I didn’t quite make it. And…​the robot rolled off of the cliff.

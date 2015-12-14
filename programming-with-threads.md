### 多线程编 

#### 线程概述
Threads Overview
(Intentionally) Missing Functionality
Implementation Decisions and Open Questions
Thread Stack Sizes
As of August 2003:
Porting Code from the Old Thread Model
Background Terminal Input
Overview
An example
A more elaborate example.
Summary
The Threads which Clozure CL Uses for Its Own Purposes
Threads Dictionary
Threads Overview

Clozure CL提供了在一个会话中执行多线程执行的功能(线程，有时也被称为轻量级进程或者就干脆叫进程，不要跟后面的系统的进程的概念混淆)。本文档描述了在Clozure CL里面多线程编程的特性以及问题。  

Clozure CL provides facilities which enable multiple threads of execution (threads, sometimes called lightweight processes or just processes, though the latter term shouldn't be confused with the OS's notion of a process) within a lisp session. This document describes those facilities and issues related to multithreaded programming in Clozure CL.  

在任何可能的地方，我都将用"thread"来表示lisp线程，尽管很多函数在他们的名字里面使用"process"。
Wherever possible, I'll try to use the term "thread" to denote a lisp thread, even though many of the functions in the API have the word "process" in their name. A lisp-process is a lisp object (of type CCL:PROCESS) which is used to control and communicate with an underlying native thread. Sometimes, the distinction between these two (quite different) objects can be blurred; other times, it's important to maintain.

Lisp threads share the same address space, but maintain their own execution context (stacks and registers) and their own dynamic binding context.

Traditionally, Clozure CL's threads have been cooperatively scheduled: through a combination of compiler and runtime support, the currently executing lisp thread arranged to be interrupted at certain discrete points in its execution (typically on entry to a function and at the beginning of any looping construct). This interrupt occurred several dozen times per second; in response, a handler function might observe that the current thread had used up its time slice and another function (the lisp scheduler) would be called to find some other thread that was in a runnable state, suspend execution of the current thread, and resume execution of the newly executed thread. The process of switching contexts between the outgoing and incoming threads happened in some mixture of Lisp and assembly language code; as far as the OS was concerned, there was one native thread running in the Lisp image and its stack pointer and other registers just happened to change from time to time.

Under Clozure CL's cooperative scheduling model, it was possible (via the use of the CCL:WITHOUT-INTERRUPTS construct) to defer handling of the periodic interrupt that invoked the lisp scheduler; it was not uncommon to use WITHOUT-INTERRUPTS to gain safe, exclusive access to global data structures. In some code (including much of Clozure CL itself) this idiom was very common: it was (justifiably) believed to be an efficient way of inhibiting the execution of other threads for a short period of time.

The timer interrupt that drove the cooperative scheduler was only able to (pseudo-)preempt lisp code: if any thread called a blocking OS I/O function, no other thread could be scheduled until that thread resumed execution of lisp code. Lisp library functions were generally attuned to this constraint, and did a complicated mixture of polling and "timed blocking" in an attempt to work around it. Needless to say, this code is complicated and less efficient than it might be; it meant that the lisp was a little busier than it should have been when it was "doing nothing" (waiting for I/O to be possible.)

For a variety of reasons - better utilization of CPU resources on single and multiprocessor systems and better integration with the OS in general - threads in Clozure CL 0.14 and later are preemptively scheduled. In this model, lisp threads are native threads and all scheduling decisions involving them are made by the OS kernel. (Those decisions might involve scheduling multiple lisp threads simultaneously on multiple processors on SMP systems.) This change has a number of subtle effects:

it is possible for two (or more) lisp threads to be executing simultaneously, possibly trying to access and/or modify the same data structures. Such access really should have been coordinated through the use of synchronization objects regardless of the scheduling modeling effect; preemptively scheduled threads increase the chance of things going wrong at the wrong time and do not offer lightweight alternatives to the use of those synchronization objects.

even on a single-processor system, a context switch can happen on any instruction boundary. Since (in general) other threads might allocate memory, this means that a GC can effectively take place at any instruction boundary. That's mostly an issue for the compiler and runtime system to be aware of, but it means that certain practices(such as trying to pass the address of a lisp object to foreign code)that were always discouraged are now discouraged ... vehemently.

there is no simple and efficient way to "inhibit the scheduler"or otherwise gain exclusive access to the entire CPU.

There are a variety of simple and efficient ways to synchronize access to particular data structures.

As a broad generalization: code that's been aggressively tuned to the constraints of the cooperative scheduler may need to be redesigned to work well with the preemptive scheduler (and code written to run under Clozure CL's interface to the native scheduler may be less portable to other CL implementations, many of which offer a cooperative scheduler and an API similar to Clozure CL (< 0.14) 's.) At the same time, there's a large overlap in functionality in the two scheduling models, and it'll hopefully be possible to write interesting and useful MP code that's largely independent of the underlying scheduling details.

The keyword :OPENMCL-NATIVE-THREADS is on *FEATURES* in 0.14 and later and can be used for conditionalization where required.

(Intentionally) Missing Functionality

Much of the functionality described above is similar to that provided by Clozure CL's cooperative scheduler, some other parts of which make no sense in a native threads implementation.

PROCESS-RUN-REASONS and PROCESS-ARREST-REASONS were SETFable process attributes; each was just a list of arbitrary tokens. A thread was eligible for scheduling (roughly equivalent to being "enabled") if its arrest-reasons list was empty and its run-reasons list was not. I don't think that it's appropriate to encourage a programming style in which otherwise runnable threads are enabled and disabled on a regular basis (it's preferable for threads to wait for some sort of synchronization event to occur if they can't occupy their time productively.)

There were a number of primitives for maintaining process queues;that's now the OS's job.

Cooperative threads were based on coroutining primitives associated with objects of type STACK-GROUP. STACK-GROUPs no longerexist.

Implementation Decisions and Open Questions

Thread Stack Sizes

When you use MAKE-PROCESS to create a thread, you can specify a stack size. Clozure CL does not impose a limit on the stack size you choose, but there is some evidence that choosing a stack size larger than the operating system's limit can cause excessive paging activity, at least on some operating systems.

The maximum stack size is operating-system-dependent. You can use shell commands to determine what it is on your platform. In bash, use "ulimit -s -H" to find the limit; in tcsh, use "limit -h s".

This issue does not affect programs that create threads using the default stack size, which you can do either by specifying no value for the :stack-size argument to MAKE-PROCESS, or by specifying the value CCL::*default-control-stack-size*.

If your program creates threads with a specified stack size, and that size is larger than the OS-specified limit, you may want to consider reducing the stack size in order to avoid possible excessive paging activity.

As of August 2003:

It's not clear that exposing PROCESS-SUSPEND/PROCESS-RESUME is a good idea: it's not clear that they offer ways to win, and it's clear that they offer ways to lose.

It has traditionally been possible to reset and enable a process that's "exhausted" . (As used here, the term "exhausted" means that the process's initial function has run and returned and the underlying native thread has been deallocated.) One of the principal uses of PROCESS-RESET is to "recycle" threads; enabling an exhausted process involves creating a new native thread (and stacks and synchronization objects and ...),and this is the sort of overhead that such a recycling scheme is seeking to avoid. It might be worth trying to tighten things up and declare that it's an error to apply PROCESS-ENABLE to an exhausted thread (and to make PROCESS-ENABLE detect this error.)

When native threads that aren't created by Clozure CL first call into lisp, a "foreign process" is created, and that process is given its own set of initial bindings and set up to look mostly like a process that had been created by MAKE-PROCESS. The life cycle of a foreign process is certainly different from that of a lisp-created one: it doesn't make sense to reset/preset/enable a foreign process, and attempts to perform these operations should be detected and treated as errors.

Porting Code from the Old Thread Model

Older versions of Clozure CL used what are often called "user-mode threads", a less versatile threading model which does not require specific support from the operating system. This section discusses how to port code which was written for that mode.

It's hard to give step-by-step instructions; there are certainly a few things that one should look at carefully:

It's wise to be suspicious of most uses of WITHOUT-INTERRUPTS; there may be exceptions, but WITHOUT-INTERRUPTS is often used as shorthand for WITH-APPROPRIATE-LOCKING. Determining what type of locking is appropriate and writing the code to implement it is likely to be straightforward and simple most of the time.

I've only seen one case where a process's "run reasons" were used to communicate information as well as to control execution; I don't think that this is a common idiom, but may be mistaken about that.

It's certainly possible that programs written for cooperatively scheduled lisps that have run reliably for a long time have done so by accident: resource-contention issues tend to be timing-sensitive, and decoupling thread scheduling from lisp program execution affects timing. I know that there is or was code in both Clozure CL and commercial MCL that was written under the explicit assumption that certain sequences of open-coded operations were uninterruptable; it's certainly possible that the same assumptions have been made (explicitly or otherwise) by application developers.

Background Terminal Input

Overview

Unless and until Clozure CL provides alternatives (via window streams, telnet streams, or some other mechanism) all lisp processes share a common *TERMINAL-IO* stream (and therefore share *DEBUG-IO*, *QUERY-IO*, and other standard and internal interactive streams.)

It's anticipated that most lisp processes other than the "Initial" process run mostly in the background. If a background process writes to the output side of *TERMINAL-IO*, that may be a little messy and a little confusing to the user, but it shouldn't really be catastrophic. All I/O to Clozure CL's buffered streams goes thru a locking mechanism that prevents the worst kinds of resource-contention problems.

Although the problems associated with terminal output from multiple processes may be mostly cosmetic, the question of which process receives input from the terminal is likely to be a great deal more important. The stream locking mechanisms can make a confusing situation even worse: competing processes may "steal" terminal input from each other unless locks are held longer than they otherwise need to be, and locks can be held longer than they need to be (as when a process is merely waiting for input to become available on an underlying file descriptor).

Even if background processes rarely need to intentionally read input from the terminal, they may still need to do so in response to errors or other unanticipated situations. There are tradeoffs involved in any solution to this problem. The protocol described below allows background processes which follow it to reliably prompt for and receive terminal input. Background processes which attempt to receive terminal input without following this protocol will likely hang indefinitely while attempting to do so. That's certainly a harsh tradeoff, but since attempts to read terminal input without following this protocol only worked some of the time anyway, it doesn't seem to be an unreasonable one.

In the solution described here (and introduced in Clozure CL 0.9), the internal stream used to provide terminal input is always locked by some process (the "owning" process.) The initial process (the process that typically runs the read-eval-print loop) owns that stream when it's first created. By using the macro WITH-TERMINAL-INPUT, background processes can temporarily obtain ownership of the terminal and relinquish ownership to the previous owner when they're done with it.

In Clozure CL, BREAK, ERROR, CERROR, Y-OR-N-P, YES-OR-NO-P, and CCL:GET-STRING- FROM-USER are all defined in terms of WITH-TERMINAL-INPUT, as are the :TTY user-interfaces to STEP and INSPECT.

An example


? Welcome to Clozure CL Version (Beta: linux) 0.9!
?


? (process-run-function "sleeper" #'(lambda () (sleep 5) (break "broken")))
#<PROCESS sleeper(1) [Enabled] #x3063B33E>


?
;;
;; Process sleeper(1) needs access to terminal input.
;;
      


This example was run under ILISP; ILISP often gets confused if one tries to enter input and "point" doesn't follow a prompt. Entering a "simple" expression at this point gets it back in synch; that's otherwise not relevant to this example.

()
NIL
? (:y 1)
;;
;; process sleeper(1) now controls terminal input
;;
> Break in process sleeper(1): broken
> While executing: #<Anonymous Function #x3063B276>
> Type :GO to continue, :POP to abort.
> If continued: Return from BREAK.
Type :? for other options.
1 > :b
(30C38E30) : 0 "Anonymous Function #x3063B276" 52
(30C38E40) : 1 "Anonymous Function #x304984A6" 376
(30C38E90) : 2 "RUN-PROCESS-INITIAL-FORM" 340
(30C38EE0) : 3 "%RUN-STACK-GROUP-FUNCTION" 768
1 > :pop
;;
;; control of terminal input restored to process Initial(0)
;;
?
      
A more elaborate example.

If a background process ("A") needs access to the terminal input stream and that stream is owned by another background process ("B"), process "A" announces that fact, then waits until the initial process regains control.


? Welcome to Clozure CL Version (Beta: linux) 0.9!
?


? (process-run-function "sleep-60" #'(lambda () (sleep 60) (break "Huh?")))
#<PROCESS sleep-60(1) [Enabled] #x3063BF26>


? (process-run-function "sleep-5" #'(lambda () (sleep 5) (break "quicker")))
#<PROCESS sleep-5(2) [Enabled] #x3063D0A6>


?       ;;
;; Process sleep-5(2) needs access to terminal input.
;;
()
NIL


? (:y 2)
;;
;; process sleep-5(2) now controls terminal input
;;
> Break in process sleep-5(2): quicker
> While executing: #x3063CFDE>
> Type :GO to continue, :POP to abort.
> If continued: Return from BREAK.
Type :? for other options.
1 >     ;; Process sleep-60(1) will need terminal access when
;; the initial process regains control of it.
;;
()
NIL
1 > :pop
;;
;; Process sleep-60(1) needs access to terminal input.
;;
;;
;; control of terminal input restored to process Initial(0)
;;


? (:y 1)
;;
;; process sleep-60(1) now controls terminal input
;;
> Break in process sleep-60(1): Huh?
> While executing: #x3063BE5E>
> Type :GO to continue, :POP to abort.
> If continued: Return from BREAK.
Type :? for other options.
1 > :pop
;;
;; control of terminal input restored to process Initial(0)
;;


?
      


Summary

This scheme is certainly not bulletproof: imaginative use of PROCESS-INTERRUPT and similar functions might be able to defeat it and deadlock the lisp, and any scenario where several background processes are clamoring for access to the shared terminal input stream at the same time is likely to be confusing and chaotic. (An alternate scheme, where the input focus was magically granted to whatever thread the user was thinking about, was considered and rejected due to technical limitations.)

The longer-term fix would probably involve using network or window-system streams to give each process unique instances of *TERMINAL-IO*.

Existing code that attempts to read from *TERMINAL-IO* from a background process will need to be changed to use WITH-TERMINAL-INPUT. Since that code was probably not working reliably in previous versions of Clozure CL, this requirement doesn't seem to be too onerous.

Note that WITH-TERMINAL-INPUT both requests ownership of the terminal input stream and promises to restore that ownership to the initial process when it's done with it. An ad hoc use of READ or READ-CHAR doesn't make this promise; this is the rationale for the restriction on the :Y command.

The Threads which Clozure CL Uses for Its Own Purposes

In the "tty world", Clozure CL starts out with 2 lisp-level threads:

? :proc
1 : -> listener     [Active]
0 :    Initial      [Active]
    
If you look at a running Clozure CL with a debugging tool, such as GDB, or Apple's Thread Viewer.app, you'll see an additional kernel-level thread on Darwin; this is used by the Mach exception-handling mechanism.

The initial thread, conveniently named "initial", is the one that was created by the operating system when it launched Clozure CL. It maps the heap image into memory, does some Lisp-level initialization, and, when the Cocoa IDE isn't being used, creates the thread "listener", which runs the top-level loop that reads input, evaluates it, and prints the result.

After the listener thread is created, the initial thread does "housekeeping": it sits in a loop, sleeping most of the time and waking up occasionally to do "periodic tasks". These tasks include forcing output on specified interactive streams, checking for and handling control-C interrupts, etc. Currently, those tasks also include polling for the exit status of external processes and handling some kinds of I/O to and from those processes.

In this environment, the initial thread does these "housekeeping" activities as necessary, until ccl:quit is called; quitting interrupts the initial thread, which then ends all other threads in as orderly a fashion as possible and calls the C function #_exit.

The short-term plan is to handle each external-process in a dedicated thread; the worst-case behavior of the current scheme can involve busy-waiting and excessive CPU utilization while waiting for an external process to terminate in some cases.

The Cocoa features use more threads. Adding a Cocoa listener creates two threads:

      ? :proc
      3 : -> Listener     [Active]
      2 :    housekeeping  [Active]
      1 :    listener     [Active]
      0 :    Initial      [Active]
    
The Cocoa event loop has to run in the initial thread; when the event loop starts up, it creates a new thread to do the "housekeeping" tasks which the initial thread would do in the terminal-only mode. The initial thread then becomes the one to receive all Cocoa events from the window server; it's the only thread which can.

It also creates one "Listener" (capital-L) thread for each listener window, with a lifetime that lasts as long as the thread does. So, if you open a second listener, you'll see five threads all together:

      ? :proc
      4 : -> Listener-2   [Active]
      3 :    Listener     [Active]
      2 :    housekeeping  [Active]
      1 :    listener     [Active]
      0 :    Initial      [Active]
    
Unix signals, such as SIGINT (control-C), invoke a handler installed by the Lisp kernel. Although the OS doesn't make any specific guarantee about which thread will receive the signal, in practice, it seems to be the initial thread. The handler just sets a flag and returns; the housekeeping thread (which may be the initial thread, if Cocoa's not being used) will check for the flag and take whatever action is appropriate to the signal.

In the case of SIGINT, the action is to enter a break loop, by calling on the thread being interrupted. When there's more than one Lisp listener active, it's not always clear what thread that should be, since it really depends on the user's intentions, which there's no way to divine programmatically. To make its best guess, the handler first checks whether the value of ccl:*interactive-abort-process* is a thread, and, if so, uses it. If that fails, it chooses the thread which currently "owns" the default terminal input stream; see .

In the bleeding-edge version of the Cocoa support which is based on Hemlock, an Emacs-like editor, each editor window has a dedicated thread associated with it. When a keypress event comes in which affects that specific window the initial thread sends it to the window's dedicated thread. The dedicated thread is responsible for trying to interpret keypresses as Hemlock commands, applying those commands to the active buffer; it repeats this in a loop, until the window closes. The initial thread handles all other events, such as mouse clicks and drags.

This thread-per-window scheme makes many things simpler, including the process of entering a "recursive command loop" in commands like "Incremental Search Forward", etc. (It might be possible to handle all Hemlock commands in the Cocoa event thread, but these "recursive command loops" would have to maintain a lot of context/state information; threads are a straightforward way of maintaining that information.)

Currently (August 2004), when a dedicated thread needs to alter the contents of the buffer or the selection, it does so by invoking methods in the initial thread, for synchronization purposes, but this is probably overkill and will likely be replaced by a more efficient scheme in the future.

The per-window thread could probably take more responsibility for drawing and handling the screen than it currently does; -something- needs to be done to buffer screen updates a bit better in some cases: you don't need to see everything that happens during something like indentation; you do need to see the results...

When Hemlock is being used, listener windows are editor windows, so in addition to each "Listener" thread, you should also see a thread which handles Hemlock command processing.

The Cocoa runtime may make additional threads in certain special situations; these threads usually don't run lisp code, and rarely if ever run much of it.

Threads Dictionary

all-processes [Function]
Returns a fresh list of all lisp processes (threads) known to Clozure CL as of the precise instant it's called. Since other threads can create and kill threads at any time, there's no way to get a perfectly accurate list of all threads.

make-process name &key persistent priority class initargs stack-size vstack-size tstack-size initial-bindings use-standard-initial-bindings [Function]
Creates and returns a new process.

name
a string, used to identify the process.

persistent
if true, requests that information about the process be retained by SAVE-APPLICATION so that an equivalent process can be restarted when a saved image is run. The default is nil.

priority
ignored. It shouldn't be ignored of course, but there are complications on some platforms. The default is 0.

class
the class of process object to create; should be a subclass of CCL:PROCESS. The default is CCL:PROCESS.

initargs
Any additional initargs to pass to MAKE-INSTANCE. The default is ().

stack-size
the size, in bytes, of the newly-created process's control stack; used for foreign function calls and to save function return address context. The default is CCL:*DEFAULT-CONTROL-STACK-SIZE*.

vstack-size
the size, in bytes, of the newly-created process's value stack; used for lisp function arguments, local variables, and other stack-allocated lisp objects. The default is CCL:*DEFAULT-VALUE-STACK-SIZE*.

tstack-size
the size, in bytes, of the newly-created process's temp stack; used for the allocation of dynamic-extent objects. The default is CCL:*DEFAULT-TEMP-STACK-SIZE*.

use-standard-initial-bindings
when true, the global "standard initial bindings" are put into effect in the new thread before. See DEF-STANDARD-INITIAL-BINDING. "standard" initial bindings are put into effect before any bindings specified by :initial-bindings are. The default is t.

This option is deprecated: the correct behavior of many Clozure CL components depends on thread-local bindings of many special variables being in effect.

initial-bindings
an alist of (symbol . valueform) pairs, which can be used to initialize special variable bindings in the new thread. Each valueform is used to compute the value of a new binding of symbol in the execution environment of the newly-created thread. The default is nil.

process
the newly-created process.

Creates and returns a new lisp process (thread) with the specified attributes. process will not begin execution immediately; it will need to be preset (given an initial function to run, as by process-preset) and enabled (allowed to execute, as by process-enable) before it's able to actually do anything.

If valueform is a function, it is called, with no arguments, in the execution environment of the newly-created thread; the primary value it returns is used for the binding of the corresponding symbol.

Otherwise, valueform is evaluated in the execution environment of the newly-created thread, and the resulting value is used.

process-preset, process-enable, process-run-function

process-suspend process [Function]
Suspends a specified process.

process
a lisp process (thread).

result
T if process had been runnable and is now suspended; NIL otherwise. That is, T if process's process-suspend-count transitioned from 0 to 1.

Suspends process, preventing it from running, and stopping it if it was already running. This is a fairly expensive operation, because it involves a few calls to the OS. It also risks creating deadlock if used improperly, for instance, if the process being suspended owns a lock or other resource which another process will wait for.

Each call to process-suspend must be reversed by a matching call to process-resume before process is able to run. What process-suspend actually does is increment the process-suspend-count of process.

A process can't suspend itself, though this once worked and this documentation claimed has claimed that it did.

process-resume, process-suspend-count

process-suspend was previously called process-disable. process-enable now names a function for which there is no obvious inverse, so process-disable is no longer defined.

process-resume process [Function]
Resumes a specified process which had previously been suspended by process-suspend.

process
a lisp process (thread).

result
T if process had been suspended and is now runnable; NIL otherwise. That is, T if process's process-suspend-count transitioned from to 0.

Undoes the effect of a previous call to process-suspend; if all such calls are undone, makes the process runnable. Has no effect if the process is not suspended. What process-resume actually does is decrement the process-suspend-count of process, to a minimum of 0.

process-suspend, process-suspend-count

This was previously called PROCESS-ENABLE; process-enable now does something slightly different.

process-suspend-count process [Function]
Returns the number of currently-pending suspensions applicable to a given process.

process
a lisp process (thread).

result
The number of "outstanding" process-suspend calls on process, or NIL if process has expired.

An "outstanding" process-suspend call is one which has not yet been reversed by a call to process-resume. A process expires when its initial function returns, although it may later be reset.

A process is runnable when it has a process-suspend-count of 0, has been preset as by process-preset, and has been enabled as by process-enable. Newly-created processes have a process-suspend-count of 0.

process-suspend, process-resume

process-preset process function &rest args [Function]
Sets the initial function and arguments of a specified process.

process
a lisp process (thread).

function
a function, designated by itself or by a symbol which names it.

args
a list of values, appropriate as arguments to function.

result
undefined.

Typically used to initialize a newly-created or newly-reset process, setting things up so that when process becomes enabled, it will begin execution by applying function to args. process-preset does not enable process, although a process must be process-preset before it can be enabled. Processes are normally enabled by process-enable.

make-process, process-enable, process-run-function

process-enable process &optional timeout [Function]
Begins executing the initial function of a specified process.

process
a lisp process (thread).

timeout
a time interval in seconds. May be any non-negative real number the floor of which fits in 32 bits. The default is 1.

result
undefined.

Tries to begin the execution of process. An error is signaled if process has never been process-preset. Otherwise, process invokes its initial function.

process-enable attempts to synchronize with process, which is presumed to be reset or in the act of resetting itself. If this attempt is not successful within the time interval specified by timeout, a continuable error is signaled, which offers the opportunity to continue waiting.

A process cannot meaningfully attempt to enable itself.

make-process, process-preset, process-run-function

It would be nice to have more discussion of what it means to synchronize with the process.

process-run-function process-specifier function &rest args [Function]
Creates a process, presets it, and enables it.

process-specifier
name | (&key namepersistentpriorityclassinitargsstack-sizevstack-sizetstack-size)

name
a string, used to identify the process. Passed to make-process.

function
a function, designated by itself or by a symbol which names it. Passed to process-preset.

persistent
a boolean, passed to make-process.

priority
ignored.

class
a subclass of CCL:PROCESS. Passed to make-process.

initargs
a list of any additional initargs to pass to make-process.

stack-size
a size, in bytes. Passed to make-process.

vstack-size
a size, in bytes. Passed to make-process.

tstack-size
a size, in bytes. Passed to make-process.

process
the newly-created process.

Creates a lisp process (thread) via make-process, presets it via process-preset, and enables it via process-enable. This means that process will immediately begin to execute. process-run-function is the simplest way to create and run a process.

make-process, process-preset, process-enable

process-interrupt process function &rest args [Function]
Arranges for the target process to invoke a specified function at some point in the near future, and then return to what it was doing.

process
a lisp process (thread).

function
a function.

args
a list of values, appropriate as arguments to function.

result
the result of applying function to args if process is the *current-process*, otherwise NIL.

Arranges for process to apply function to args at some point in the near future (interrupting whatever process was doing.) If function returns normally, process resumes execution at the point at which it was interrupted.

process must be in an enabled state in order to respond to a process-interrupt request. It's perfectly legal for a process to call process-interrupt on itself.

process-interrupt uses asynchronous POSIX signals to interrupt threads. If the thread being interrupted is executing lisp code, it can respond to the interrupt almost immediately (as soon as it has finished pseudo-atomic operations like consing and stack-frame initialization.)

If the interrupted thread is blocking in a system call, that system call is aborted by the signal and the interrupt is handled on return.

It is still difficult to reliably interrupt arbitrary foreign code (that may be stateful or otherwise non-reentrant); the interrupt request is handled when such foreign code returns to or enters lisp.

without-interrupts

It would probably be better for result to always be NIL, since the present behavior is inconsistent.

process-interrupt works by sending signals between threads, via the C function #_pthread_signal. It could be argued that it should be done in one of several possible other ways under Darwin, to make it practical to asynchronously interrupt things which make heavy use of the Mach nanokernel.

*current-process* [Variable]
Bound separately in each process, to that process itself. It may be used when lisp code needs to find out what process it is executing in. It should not be set by user code.

process-reset process &optional kill-option [Function]
This function causes process to cleanly exit from any ongoing computation and enter a state wehre it can be process-preset.

This is implemented by signaling a condition of type PROCESS-RESET; user-defined condition handlers should generally refrain from attempting to handle conditions of this type.

The kill-option argument is for internal use only and should not be specified by user code.

A process can meaningfully reset itself.

There is in general no way to know precisely when process has completed the act of resetting or killing itself; a process which has either entered the limbo of the reset state or exited has few ways of communicating either fact.

The function process-enable can reliably determine when a process has entered the limbo of the reset state, but can't predict how long the clean exit from ongoing computation might take: that depends on the behavior of unwind-protect cleanup forms, and of the OS scheduler.

Resetting a process other than *current-process* involves the use of the function process-interrupt.

process-reset-and-enable process [Function]
Reset and enable the specified process, which may not be the current process.

process
a lisp process (thread), which may not be the current process.

result
undefined.

Equivalent to calling (process-reset process) and (process-enable process).

process-reset, process-enable

process-kill process [Function]
Causes process to cleanly exit from any ongoing computation, and then exit. Note that unwind-protect cleanup forms will be run with interrupts disabled.

process-abort process &optional condition [Function]
Causes a specified process to process an abort condition, as if it had invoked abort.

process
a lisp process (thread).

condition
a lisp condition. The default is NIL.

Entirely equivalent to calling (process-interruptprocess (lambda () (abortcondition))). Causes process to transfer control to the applicable handler or restart for abort.

If condition is non-NIL, process-abort does not consider any handlers which are explicitly bound to conditions other than condition.

process-reset, process-kill

*ticks-per-second* [Variable]
Bound to the clock resolution of the OS scheduler.

A positive integer.

The clock resolution of the OS scheduler. Currently, both LinuxPPC and DarwinPPC yield an initial value of 100.

This value is ordinarily of marginal interest at best, but, for backward compatibility, some functions accept timeout values expressed in "ticks". This value gives the number of ticks per second.

process-wait-with-timeout

process-whostate process [Function]
Returns a string which describes the status of a specified process.

process
a lisp process (thread).

whostate
a string which describes the "state" of process.

This information is primarily for the benefit of debugging tools. whostate is a terse report on what process is doing, or not doing, and why.

If the process is currently waiting in a call to process-wait or process-wait-with-timeout, its process-whostate will be the value which was passed to that function as whostate.

process-wait, process-wait-with-timeout, with-terminal-input

This should arguably be SETFable, but doesn't seem to ever have been.

process-allow-schedule [Function]
Used for cooperative multitasking; probably never necessary.

Advises the OS scheduler that the current thread has nothing useful to do and that it should try to find some other thread to schedule in its place. There's almost always a better alternative, such as waiting for some specific event to occur. For example, you could use a lock or semaphore.

make-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

This is a holdover from the days of cooperative multitasking. All modern general-purpose operating systems use preemptive multitasking.

process-wait whostate function &rest args [Function]
Causes the current lisp process (thread) to wait for a given predicate to return true.

whostate
a string, which will be the value of process-whostate while the process is waiting.

function
a function, designated by itself or by a symbol which names it.

args
a list of values, appropriate as arguments to function.

result
NIL.

Causes the current lisp process (thread) to repeatedly apply function to args until the call returns a true result, then returns NIL. After each failed call, yields the CPU as if by process-allow-schedule.

As with process-allow-schedule, it's almost always more efficient to wait for some specific event to occur; this isn't exactly busy-waiting, but the OS scheduler can do a better job of scheduling if it's given the relevant information. For example, you could use a lock or semaphore.

process-whostate, process-wait-with-timeout, make-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

process-wait-with-timeout whostate ticks function args [Function]
Causes the current thread to wait for a given predicate to return true, or for a timeout to expire.

whostate
a string, which will be the value of process-whostate while the process is waiting.

ticks
either a positive integer expressing a duration in "ticks" (see *ticks-per-second*), or NIL.

function
a function, designated by itself or by a symbol which names it.

args
a list of values, appropriate as arguments to function.

result
T if process-wait-with-timeout returned because its function returned true, or NIL if it returned because the duration ticks has been exceeded.

If ticks is NIL, behaves exactly like process-wait, except for returning T. Otherwise, function will be tested repeatedly, in the same kind of test/yield loop as in process-wait until either function returns true, or the duration ticks has been exceeded.

Having already read the descriptions of process-allow-schedule and process-wait, the astute reader has no doubt anticipated the observation that better alternatives should be used whenever possible.

*ticks-per-second*, process-whostate, process-wait, make-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

without-interrupts &body body [Macro]
Evaluates its body in an environment in which process-interrupt requests are deferred.

body
an implicit progn.

result
the primary value returned by body.

Executes body in an environment in which process-interrupt requests are deferred. As noted in the description of process-interrupt, this has nothing to do with the scheduling of other threads; it may be necessary to inhibit process-interrupt handling when (for instance) modifying some data structure (for which the current thread holds an appropriate lock) in some manner that's not reentrant.

process-interrupt

with-interrupts-enabled &body body [Macro]
Evaluates its body in an environment in which process-interrupt requests have immediate effect.

body
an implicit progn.

result
the primary value returned by body.

Executes body in an environment in which process-interrupt requests have immediate effect.

make-lock &optional name [Function]
Creates and returns a lock object, which can be used for synchronization between threads.

name
any lisp object; saved as part of lock. Typically a string or symbol which may appear in the process-whostates of threads which are waiting for lock.

lock
a newly-allocated object of type CCL:LOCK.

Creates and returns a lock object, which can be used to synchronize access to some shared resource. lock is initially in a "free" state; a lock can also be "owned" by a thread.

with-lock-grabbed, grab-lock, release-lock, try-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

with-lock-grabbed (lock) &body body [Macro]
Waits until a given lock can be obtained, then evaluates its body with the lock held.

lock
an object of type CCL:LOCK.

body
an implicit progn.

result
the primary value returned by body.

Waits until lock is either free or owned by the calling thread, then executes body with the lock owned by the calling thread. If lock was free when with-lock-grabbed was called, it is restored to a free state after body is executed.

make-lock, grab-lock, release-lock, try-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

grab-lock lock [Function]
Waits until a given lock can be obtained, then obtains it.

lock
an object of type CCL:LOCK.

Blocks until lock is owned by the calling thread.

The macro with-lock-grabbedcould be defined in terms of grab-lock and release-lock, but it is actually implemented at a slightly lower level.

make-lock, with-lock-grabbed, release-lock, try-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

release-lock lock [Function]
Relinquishes ownership of a given lock.

lock
an object of type CCL:LOCK.

Signals an error of type CCL:LOCK-NOT-OWNER if lock is not already owned by the calling thread; otherwise, undoes the effect of one previous grab-lock. If this means that release-lock has now been called on lock the same number of times as grab-lock has, lock becomes free.

make-lock, with-lock-grabbed, grab-lock, try-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

try-lock lock [Function]
Obtains the given lock, but only if it is not necessary to wait for it.

lock
an object of type CCL:LOCK.

result
T if lock has been obtained, or NIL if it has not.

Tests whether lock can be obtained without blocking - that is, either lock is already free, or it is already owned by *current-process*. If it can, causes it to be owned by the calling lisp process (thread) and returns T. Otherwise, the lock is already owned by another thread and cannot be obtained without blocking; NIL is returned in this case.

make-lock, with-lock-grabbed, grab-lock, release-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

make-read-write-lock-write-lock [Function]
Creates and returns a read-write lock, which can be used for synchronization between threads.

read-write-lock
a newly-allocated object of type CCL:READ-WRITE-LOCK.

Creates and returns an object of type CCL::READ-WRITE-LOCK. A read-write lock may, at any given time, belong to any number of lisp processes (threads) which act as "readers"; or, it may belong to at most one process which acts as a "writer". A read-write lock may never be held by a reader at the same time as a writer. Initially, read-write-lock has no readers and no writers.

with-read-lock, with-write-lock, make-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

There probably should be some way to atomically "promote" a reader, making it a writer without releasing the lock, which could otherwise cause delay.

with-read-lock (read-write-lock) &body body [Macro]
Waits until a given lock is available for read-only access, then evaluates its body with the lock held.

read-write-lock
an object of type CCL:READ-WRITE-LOCK.

body
an implicit progn.

result
the primary value returned by body.

Waits until read-write-lock has no writer, ensures that *current-process* is a reader of it, then executes body.

After executing body, if *current-process* was not a reader of read-write-lock before with-read-lock was called, the lock is released. If it was already a reader, it remains one.

make-read-write-lock, with-write-lock, make-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

with-write-lock (read-write-lock) &body body [Macro]
Waits until the given lock is available for write access, then executes its body with the lock held.

read-write-lock
an object of type CCL:READ-WRITE-LOCK.

body
an implicit progn.

result
the primary value returned by body.

Waits until read-write-lock has no readers and no writer other than *current-process*, then ensures that *current-process* is the writer of it. With the lock held, executes body.

After executing body, if *current-process* was not the writer of read-write-lock before with-write-lock was called, the lock is released. If it was already the writer, it remains the writer.

make-read-write-lock, with-read-lock, make-lock, make-semaphore, process-input-wait, process-output-wait, with-terminal-input

make-semaphore [Function]
Creates and returns a semaphore, which can be used for synchronization between threads.

semaphore
a newly-allocated object of type CCL:SEMAPHORE.

Creates and returns an object of type CCL:SEMAPHORE. A semaphore has an associated "count" which may be incremented and decremented atomically; incrementing it represents sending a signal, and decrementing it represents handling that signal. semaphore has an initial count of 0.

signal-semaphore, wait-on-semaphore, timed-wait-on-semaphore, make-lock, make-read-write-lock, process-input-wait, process-output-wait, with-terminal-input

signal-semaphore semaphore [Function]
Atomically increments the count of a given semaphore.

semaphore
an object of type CCL:SEMAPHORE.

result
an integer representing an error identifier which was returned by the underlying OS call.

Atomically increments semaphore's "count" by 1; this may enable a waiting thread to resume execution.

make-semaphore, wait-on-semaphore, timed-wait-on-semaphore, make-lock, make-read-write-lock, process-input-wait, process-output-wait, with-terminal-input

result should probably be interpreted and acted on by signal-semaphore, because it is not likely to be meaningful to a lisp program, and the most common cause of failure is a type error.

wait-on-semaphore semaphore [Function]
Waits until the given semaphore has a positive count which can be atomically decremented.

semaphore
an object of type CCL:SEMAPHORE.

result
an integer representing an error identifier which was returned by the underlying OS call.

Waits until semaphore has a positive count that can be atomically decremented; this will succeed exactly once for each corresponding call to SIGNAL-SEMAPHORE.

make-semaphore, signal-semaphore, timed-wait-on-semaphore, make-lock, make-read-write-lock, process-input-wait, process-output-wait, with-terminal-input

result should probably be interpreted and acted on by wait-on-semaphore, because it is not likely to be meaningful to a lisp program, and the most common cause of failure is a type error.

timed-wait-on-semaphore semaphore timeout [Function]
Waits until the given semaphore has a positive count which can be atomically decremented, or until a timeout expires.

semaphore
An object of type CCL:SEMAPHORE.

timeout
a time interval in seconds. May be any non-negative real number the floor of which fits in 32 bits. The default is 1.

result
T if timed-wait-on-semaphore returned because it was able to decrement the count of semaphore; NIL if it returned because the duration timeout has been exceeded.

Waits until semaphore has a positive count that can be atomically decremented, or until the duration timeout has elapsed.

make-semaphore, wait-on-semaphore, make-lock, make-read-write-lock, process-input-wait, process-output-wait, with-terminal-input

process-input-wait fd &optional timeout [Function]
Waits until input is available on a given file-descriptor.

fd
a file descriptor, which is a non-negative integer used by the OS to refer to an open file, socket, or similar I/O connection. See stream-device.

timeout
either NIL or a time interval in milliseconds. Must be a non-negative integer. The default is NIL.

Wait until input is available on fd. This uses the select() system call, and is generally a fairly efficient way of blocking while waiting for input. More accurately, process-input-wait waits until it's possible to read from fd without blocking, or until timeout, if it is not NIL, has been exceeded.

Note that it's possible to read without blocking if the file is at its end - although, of course, the read will return zero bytes.

make-lock, make-read-write-lock, make-semaphore, process-output-wait, with-terminal-input

process-input-wait has a timeout parameter, and process-output-wait does not. This inconsistency should probably be corrected.

process-output-wait fd &optional timeout [Function]
Waits until output is possible on a given file descriptor.

fd
a file descriptor, which is a non-negative integer used by the OS to refer to an open file, socket, or similar I/O connection. See stream-device.

timeout
either NIL or a time interval in milliseconds. Must be a non-negative integer. The default is NIL.

Wait until output is possible on fd or until timeout, if it is not NIL, has been exceeded. This uses the select() system call, and is generally a fairly efficient way of blocking while waiting to output.

If process-output-wait is called on a network socket which has not yet established a connection, it will wait until the connection is established. This is an important use, often overlooked.

make-lock, make-read-write-lock, make-semaphore, process-input-wait, with-terminal-input

process-input-wait has a timeout parameter, and process-output-wait does not. This inconsistency should probably be corrected.

with-terminal-input &body body [Macro]
Executes its body in an environment with exclusive read access to the terminal.

body
an implicit progn.

result
the primary value returned by body.

Requests exclusive read access to the standard terminal stream, *terminal-io*. Executes body in an environment with that access.

*request-terminal-input-via-break*, ":Y", make-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait

*request-terminal-input-via-break* [Variable]
Controls how attempts to obtain ownership of terminal input are made.

A boolean.

NIL.

Controls how attempts to obtain ownership of terminal input are made. When NIL, a message is printed on *TERMINAL-IO*; it's expected that the user will later yield control of the terminal via the :Y toplevel command. When T, a BREAK condition is signaled in the owning process; continuing from the break loop will yield the terminal to the requesting process (unless the :Y command was already used to do so in the break loop.)

with-terminal-input, ":Y", make-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait

( :y p) [Toplevel Command]
Yields control of terminal input to a specified lisp process (thread).

p
a lisp process (thread), designated either by an integer which matches its process-serial-number, or by a string which is equal to its process-name.

:Y is a toplevel command, not a function. As such, it can only be used interactively, and only from the initial process.

The command yields control of terminal input to the process p, which must have used with-terminal-input to request access to the terminal input stream.

with-terminal-input, *request-terminal-input-via-break*, make-lock, make-read-write-lock, make-semaphore, process-input-wait, process-output-wait

join-process process &optional default [Function]
Waits for a specified process to complete and returns the values that that process's initial function returned.

process
a process, typically created by process-run-function or by make-process

default
A default value to be returned if the specified process doesn't exit normally.

values
The values returned by the specified process's initial function if that function returns, or the value of the default argument, otherwise.

Waits for the specified process to terminate. If the process terminates "normally" (if its initial function returns), returns the values that that initial function returnes. If the process does not terminate normally (e.g., if it's terminated via process-kill and a default argument is provided, returns the value of that default argument. If the process doesn't terminate normally and no default argument is provided, signals an error.

A process can't successfully join itself, and only one process can successfully receive notification of another process's termination.
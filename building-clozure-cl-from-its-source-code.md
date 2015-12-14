Building Clozure CL from its Source Code

Building Definitions
Reasons for Building
Kernel Build Prerequisites
Building Everything
Building the Kernel
Building the Heap Image
Compile Lisp Source Code
Create a Bootstrapping Image
Building a full image from a bootstrapping image
Building Definitions

The following terms are used in subsequent sections; it may be helpful to refer to these definitions.

A fasl file is the file produced by compile-file. These files store the machine code associated with function definitions and the external representation of other lisp objects in a compact, machine-readable form. Short for “FASt Loading”. Clozure CL uses different pathname types (extensions) to name fasl files on different platforms; see Platform-specific filename conventions for the details.

The lisp kernel is a C program with a fair amount of platform-specific assembly language code. The lisp kernel provides runtime support for lisp code, such as garbage collection, memory allocation, exception handling, and so forth. When the lisp kernel starts, it maps the heap image into memory and transfers control to compiled lisp code that the image contains.

A heap image is a file that can be quickly mapped into a process's address space. Conceptually, it's not too different from an executable file or shared library in the OS's native format (ELF or Mach-O/dyld format); for historical reasons, Clozure CL's own heap images are in their own (fairly simple) format. A full heap image contains all of the code and data that comprise Clozure CL. The default image file name is the name of the lisp kernel with a .image suffix. See Platform-specific filename conventions.

A bootstrapping image (see Create a Bootstrapping Image) is a minimal heap image used in the process of building Clozure CL itself. The bootstrapping image contains just enough code to load the rest of Clozure CL from fasl files. The function rebuild-ccl automatically creates a bootstrepping image as part of its work.

Each supported platform (and possibly a few as-yet-unsupported ones) has a uniquely named subdirectory of ccl/lisp-kernel/; each such kernel build directory contains a Makefile and may contain some auxiliary files (linker scripts, etc.) that are used to build the lisp kernel on a particular platform. The platform-specific name of the kernel build directory is described in Platform-specific filename conventions.

Platform-specific filename conventions
Platform	kernel	fasl extension	kernel build directory
Linux (ppc)	ppccl	.pfsl	linuxppc
Linux (ppc64)	ppccl64	.p64fsl	linuxppc64
Linux (x8664)	lx86cl64	.lx64fsl	linuxx8664
Linux (x8632)	lx86cl	.lx32fsl	linuxx8632
Linux (arm32)	armcl	.lafsl	linuxarm
OS X (x8664)	dx86cl64	.dx64fsl	darwinx8664
OS X (x8632)	dx86cl	.dx32fsl	darwinx8632
FreeBSD (x8664)	fx86cl64	.fx64fsl	freebsdx8664
FreeBSD (x8632)	fx86cl	.fx32fsl	freebsdx8632
Solaris (x8664)	sx86cl64	.sx64fsl	solarisx64
Solaris (x8632)	sx86cl	.sx32fsl	solarisx86
Windows (x8664)	wx86cl64.exe	.wx64fsl	win64
Windows (x8632)	wx86cl.exe	.wx32fsl	win32
Reasons for Building

At a given time, there are generally two versions of Clozure CL that you might want to use (and therefore might want to build from source).

The first of these is the current release branch. Fixes for serious bugs are sometimes checked into the release branch, and you might want to update from Subversion and rebuild in order to pick up these fixes.

You may also be interested in building the development version of Clozure CL, which is often called the trunk. The trunk may contain both interesting new features and interesting new bugs. See http://trac.clozure.com/ccl/wiki for information about how to check out a copy of the trunk.

Kernel Build Prerequisites

In order to build the lisp kernel, you must have installed a C compiler and its associated tools (such as as and ld). Additionally, the lisp kernel build process uses m4; this program is often not installed by default, so make sure you have it. The lisp kernel makefiles generally assume that you are using GNU make.

On a recent Mac OS X system, typing xcode-select --install should download and install everything needed to build Clozure CL.

Building Everything

With all the pre-requisites in place, building Clozure CL is very simple.

$ ccl -n
? (ccl:rebuild-ccl :full t)
This peforms the following steps:

Deletes all fasl files and other object files in the ccl directory tree

Does (compile-ccl t) in the running lisp, to generate fasl files from the lisp sources.

Does (xload-level-0 :force) in the running lisp. This compiles the lisp files in the ccl:level-0; directory and then creates a special bootstrapping image from the compiled fasl files.

Runs an external process that runs make in the current platform's kernel build directory to create a new kernel. This step can only work if the C compiler and related tools are installed; see Kernel Build Prerequisites.

Runs another external process, which causes the newly compiled lisp kernel to load the new bootstrapping image. The bootstrapping image then loads the rest of the fasl files and a new copy of the platform's full heap image is then saved.

When all goes well, this all happen without user intervention and with some simple progress messages. If anything goes wrong during execution of either of the external processes, the process output is displayed as part of a lisp error message.

rebuild-ccl is essentially just a short cut for running all the individual steps involved in rebuilding the system. You can also execute these steps individually, as described below.

Building the Kernel

Rebuilding the lisp kernel is straightfoward. Consult the table Platform-specific filename conventions to determine the name of the lisp kernel build directory you want. Then, change to that directory and say make. Suppose you wanted to build the lisp kernel for 64-bit FreeBSD/x86. We find that the name of the lisp kernel build directory is freebsdx8664, so we do the following:

$ cd ccl/lisp-kernel/freebsdx8664
$ make clean
$ make
The most common reason that the build fails is because m4 is not installed. If you see m4: command not found, then you should install m4.

On Mac OS X, make sure that you have installed the command-line developer tools with xcode-select --install. If you get an error message saying that the file sys/signal.h cannot be found, this is a sign that you need to do this. You need to do this even if you have Xcode already installed.

Building the Heap Image

Typically, rebuild-ccl is used to rebuild all of Clozure CL. In special cases, you might want to exercise more control over how the heap image is built. These cases typically arise only when developing Clozure CL itself.

Creating a new Clozure CL heap image consists of the following steps:

Using an existing lisp, recompile the updated lisp source code

Using an existing lisp, create a new bootstrapping image

Start up the lisp kernel and tell it to load the bootstrapping image you just created

Compile Lisp Source Code

Calling:

? (ccl:compile-ccl)
at the lisp prompt compiles any fasl files that are out-of-date with respect to the corresponding lisp sources; (ccl:compile-ccl t) forces recompilation. ccl:compile-ccl reloads newly-compiled versions of some files; ccl:xcompile-ccl is analogous, but skips this reloading step.

Unless there are bootstrapping considerations involved, it usually doesn't matter whether these files are reloaded after they're recompiled.

Calling compile-ccl or xcompile-ccl in an environment where fasl files don't yet exist may produce warnings to that effect whenever files are required during compilation; those warnings can be safely ignored. Depending on the maturity of the Clozure CL release, calling compile-ccl or xcompile-ccl may also produce several warnings about undefined functions, etc. They should be cleaned up at some point.

Create a Bootstrapping Image

The bootstrapping image isn't provided in Clozure CL distributions. It can be built from the source code provided in distributions (using a lisp image and kernel provided in those distributions) using the procedure described below.

The bootstrapping image is built by invoking a special utility inside a running Clozure CL heap image to load files contained in the ccl/level-0 directory. The bootstrapping image loads several dozen fasl files. After it's done so, it saves a heap image via save-application. This process is called cross-dumping.

Given a source distribution, a lisp kernel, and a heap image, one can produce a bootstrapping image by first invoking Clozure CL from the shell:

$ ccl
? (ccl:xload-level-0)
This function compiles the lisp sources in the ccl/level-0 directory as needed, and then loads the resulting fasl files into a simulated lisp heap contained in data structures inside the running lisp. It then writes this data to disk as a bootstrapping image and displays the pathname of the newly-written image on the terminal.

xload-level-0 should be called whenever your existing boot image is out-of-date with respect to the source files in ccl:level-0;.

Building a full image from a bootstrapping image

To build a full image from a bootstrapping image, just invoke the kernel and tell it to load the bootstrapping image (as reported by xload-level-0. For example, suppose you are using 64-bit Mac OS X:

$ cd ccl    # wherever your ccl directory is
$ ./dx86cl64 --image-name x86-boot64.image --no-init
Other platoforms use analogous steps: use the appropriate platform-specific name for the lisp kernel, and use the name of boot image as reported by xload-level-0.

This process will load a few dozen fasl files, printing a message as each file is loaded. If all of these files successfully load, the lisp will print a prompt. You should be able to do essentially everything in that environment that you can in the environment provided by a full heap image. If everything went well, you can save that image using save-application:

? (ccl:save-application "new.image")
The name new.image can be whatever you want. You may wish to use the default image name for your platform; see Platform-specific filename conventions.

If things go wrong in the early stages of the loading sequence, errors are often difficult to debug; until a fair amount of code (CLOS, the CL condition system, streams, the reader, the read-eval-print loop) is loaded, it's generally not possible for the lisp to report an error. Errors that occur during these early stages (“the cold load”) sometimes cause the lisp kernel debugger to be invoked; it's primitive, but can sometimes help one to get oriented.




Installing and Running Clozure CL

Installing
Running Clozure CL
The Init File
Command Line Options
Running Clozure CL as a Mac Application
Installing

Clozure CL is distributed via the Internet. Please see http://ccl.clozure.com/download.html for instructions on how to download it.

After following the download instructions, you should have a directory on your system named ccl. This directory is called the ccl directory.

Clozure CL is made up of two parts: the lisp kernel, and a heap image. When the lisp kernel starts up, it locates the heap image, maps it into memory, and starts running the lisp code contained in the image. In the ccl directory, you will find pre-built lisp kernel executables and heap images for your platform.

The names used for the lisp kernel on the various platforms are listed in the table below. The heap images have the same basename as the corresponding lisp kernel, but a .image suffix. Thus, the image name for armcl would be armcl.image.

Kernel Names by Platform
Platform	Kernel
Linux x86, x86-64	lx86cl, lx86cl64
OS X x86, x86-64	dx86cl, dx86cl64
FreeBSD x86, x86-64	fx86cl, fx86cl64
Solaris x86, x86-64	sx86cl, sx86cl64
Windows x86, x86-64	wx86cl, wx86cl64
Linux PowerPC 32-bit, 64-bit	ppccl, ppccl64
Linux ARM 32-bit (armv6)	armcl
By default, the lisp kernel will look for a heap image with an appropriate name in the same directory that the lisp kernel itself is in. Thus, it is possible to start Clozure CL simply by running ./lx86cl64 (or whatever the appropriate binary is called) directly from the ccl directory.

If the lisp kernel binary does not work, you may need to recompile it on your local system. See Building the Kernel.

Running Clozure CL

If you always run Clozure CL from Emacs, it is sufficient to use the full pathname of the lisp kernel binary directly. That is, in your Emacs init file, you could write something like (setq inferior-lisp-program "/path/to/ccl/lx86cl64") or make the equivalent changes to slime-lisp-implementations.

It can also be handy to run Clozure CL straight from a terminal prompt. In the scripts/ directory of the ccl directory, there are two files named ccl and ccl64. Copy these files into /usr/local/bin or some other directory that is on your path, and then edit them so that the value of CCL_DEFAULT_DIRECTORY is your ccl directory. You can then start up the lisp by typing ccl or ccl64.

You may wish to install scripts/ccl64 with the name ccl if you use the 64-bit lisp more. If you want the 32-bit lisp to be available as well, you can install scripts/ccl as ccl32. Note that there is nothing magical about these scripts. You should feel free to edit them as desired.

The Init File

By default, Clozure CL will look for a file named ccl-init.lisp in your home directory, and load it upon startup. On Unix systems, it will also look for .ccl-init.lisp.

If you wish, you can compile your init file, and Clozure CL will load the compiled version if it is newer than the corresponding source file. In other words, Clozure CL loads your init file with (load "home:ccl-init").

Because the init file is loaded the same way as normal Lisp code is, you can put anything you want in it. For example, you can change the working directory, and load code that you use frequently.

To suppress the loading of this init-file, invoke Clozure CL with the --no-init (or -n) option.

Command Line Options

When using Clozure CL from the command line, the following options may be used to modify its behavior. The exact set of Clozure CL command-line arguments may vary per platform and may change over time. The definitive list of command line options may be retrieved by using the --help option.

-h (or --help). Provides a definitive (if somewhat terse) summary of the command line options accepted by the Clozure CL implementation and then exits.

-V (or --version). Prints the version of Clozure CL then exits. The version string is the same value that is returned by lisp-implementation-version.

-K character-encoding-name (or --terminal-encodingcharacter-encoding-name). Specifies the character encoding to use for *terminal-io* (see Character Encodings). Specifically, the character-encoding-name string is uppercased and interned in the keyword package. If an encoding named by that keyword exists, *terminal-character-encoding-name* is set to the name of that encoding. The default us :utf-8.

-n (or --no-init). If this option is given, the init file is not loaded. This is useful if Clozure CL is being invoked by a shell script that should not be affected by whatever customizations a user might have in place.

-e form (or --eval). An expression is read (via read-from-string) from the string form and evaluated. If form contains shell metacharacters, it may be necessary to escape or quote them to prevent the shell from interpreting them.

-l path (or --load path). Loads file specified by path.

-T n (or --set-lisp-heap-gc-threshold n). Sets the Lisp gc threshold to n. (see GC Page reclamation policy

-Q (or --quiet). Suppresses printing of heralds and prompts when the --batch command line option is specified.

-R n (or --heap-reserve). Reserves n bytes for heap expansion. The default is 549755813888. (see Heap space allocation)

-S n (or --stack-sizen). Sets the size of the initial control stack to n. (see Thread Stack Sizes)

-Z n (or --thread-stack-sizen). Sets the size of the first thread's stack to n. (see Thread Stack Sizes)

-b (or --batch). Execute in batch mode. End-of-file from *standard-input* causes Clozure CL to exit, as do attempts to enter a break loop.

--no-sigtrap An obscure option for running under GDB.

-I image-name (or --image-name image-name). Specifies the image name for the kernel to load. Defaults to the kernel name with the suffix .image appended.

The --load and --eval options can each be provided multiple times. They're executed in the order specified on the command line, after the init file (if there is one) is loaded and before the toplevel read-eval-print loop is entered.

Finally, any arguments following the pseudo-argument -- are not processed, and are made available to Lisp as the value of *unprocessed-command-line-arguments*.

Running Clozure CL as a Mac Application

If you want to run Clozure CL as a double-clickable Macintosh application, you can do that. A version of Clozure CL is available from the Mac App Store if you would like to obtain it from there. Alternatively you can build the IDE yourself: please see Building the IDE. Currently, it's not possible to use the Mac App Store version of Clozure CL as a command-line program.


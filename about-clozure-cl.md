## 关于Clozure CL

### [Clozure CL介绍](#introduction-to-clozure-cl)

#### <span id="introduction-to-clozure-cl">Clozure CL介绍</span>  


Clozure CL是一个快速的、成熟的、开源的Common Lisp实现，目前支持Linux、Mac OS X、FreeBSD以及Windows平台。Clozure CL在1998年从Macintosh Common Lisp(MCL)分出来，并且开发从那时候开始完全独立。<br />
1998年当它分出来的时候命名为OpenMCL；随后，Clozure重命名为Clozure CL,部分原因是因为它的祖先MCL已经开源。如果有两个独立的开源项目名字相近，Clozure认为这个会让用户混淆；新的名字同样反映了作为Clozure联营公司的旗舰产品的Clozure CL的当前状态。  
而且，新的名字也反映了Clozure CL的起源：早期，MCL被称为Coral Common Lisp或者“CCL”。包包括Clozure CL的实现已经被称为“CCL”，缩写代表Lisp产品。

Furthermore, the new name refers to Clozure CL's ancestry: in its early years, MCL was known as Coral Common Lisp, or “CCL”. For years the package that contains most of Clozure CL's implementation-specific symbols has been named “CCL”, an acronym that once stood for the name of the Lisp product. It seems fitting that “CCL” once again stands for the name of the product.

某些命令以及源文件可能仍然用“OpenMCL”来代替Clozure CL。
Some commands and source files may still refer to “OpenMCL” instead of Clozure CL.

Clozure CL compiles to native code and supports multithreading using native OS threads. It includes a foreign-function interface, and supports both Lisp code that calls external code, and external code that calls Lisp code. Clozure CL can create standalone executables on all supported platforms.

On Mac OS X, Clozure CL supports building GUI applications that use OS X's native Cocoa frameworks, and the OS X distributions include an IDE written with Cocoa, and distributed with complete sources.

On all supported platforms, Clozure CL can run as a command-line process, or as an inferior Emacs process using either SLIME or ILISP.

Features of Clozure CL include

Very fast compilation speed.

A fast, precise, compacting, generational garbage collector written in hand-optimized C. The sizes of the generations are fully configurable. Typically, a generation can be collected in a millisecond on modern systems.

Fast execution speed, competitive with other Common Lisp implementations on most benchmarks.

Robust and stable. Customers report that their CPU-intensive, multi-threaded applications run for extended periods on Clozure CL without difficulty.

Full native OS threads on all platforms. Threads are automatically distributed across multiple cores. The API includes support for shared memory, locking, and blocking for OS operations such as I/O.

Full Unicode support.

Full SLIME integration.

An IDE on Mac OS X, fully integrated with the Macintosh window system and User Interface standards.

Excellent debugging facilities. The names of all local variables are available in a backtrace.

A complete, mature foreign function interface, including a powerful bridge to Objective-C and Cocoa on Mac OS X.

Many extensions including: files mapped to Common Lisp vectors for fast file I/O; thread-local hash tables and streams to eliminate locking overhead; cons hashing support; and much more

Very efficient use of memory

Although it's an open-source project, available free of charge under a liberal license, Clozure CL is also a fully-supported product of Clozure Associates. Clozure continues to extend, improve, and develop Clozure CL in response to customer and user needs, and offers full support and development services for Clozure CL.


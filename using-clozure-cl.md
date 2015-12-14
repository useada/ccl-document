Using Clozure CL

Introduction
Memory-mapped Files
Static Variables
Saving Applications
Concatenating FASL Files
Floating Point Numbers
Code Coverage
Overview
Limitations
Usage
Functions and Variables
Interpreting Code Coloring
Other Extensions
Introduction

The Common Lisp standard allows considerable latitude in the details of an implementation, and each particular Common Lisp system has some idiosyncrasies. This chapter describes ordinary user-level features of Clozure CL, including features that may be part of the Common Lisp standard, but which may have quirks or details in the Clozure CL implementation that are not described by the standard. It also describes extensions to the standard; that is, features of Clozure CL that are not part of the Common Lisp standard at all.

Memory-mapped Files

In release 1.2 and later, Clozure CL supports memory-mapped files. On operating systems that support memory-mapped files (including Mac OS X, Linux, and FreeBSD), the operating system can arrange for a range of virtual memory addresses to refer to the contents of an open file. As long as the file remains open, programs can read values from the file by reading addresses in the mapped range.

Using memory-mapped files may in some cases be more efficient than reading the contents of a file into a data structure in memory.

Clozure CL provides the functions map-file-to-ivector and map-file-to-octet-vector to support memory-mapping. These functions return vectors whose contents are the contents of memory-mapped files. Reading an element of such a vector returns data from the corresponding position in the file.

Without memory-mapped files, a common idiom for reading the contents of files might be something like this:

(let* ((stream (open pathname :direction :input :element-type '(unsigned-byte 8)))
       (vector (make-array (file-size-to-vector-size stream)
                           :element-type '(unsigned-byte 8))))
  (read-sequence vector stream))
    
Using a memory-mapped files has a result that is the same in that, like the above example, it returns a vector whose contents are the same as the contents of the file. It differs in that the above example creates a new vector in memory and copies the file's contents into it; using a memory-mapped file instead arranges for the vector's elements to point to the file's contents on disk directly, without copying them into memory first.

The vectors returned by map-file-to-ivector and map-file-to-octet-vector are read-only; any attempt to change an element of a vector returned by these functions results in a memory-access error. Clozure CL does not currently support writing data to memory-mapped files.

Vectors created by map-file-to-ivector and map-file-to-octet-vector are required to respect Clozure CL's limit on the total size of an array. That means that you cannot use these functions to create a vector longer than array-total-size-limit, even if the filesystem supports file sizes that are larger. The value of array-total-size-limit is (expt 2 24) on 32-but platforms; and (expt 2 56) on 64-bit platforms.

map-file-to-ivector pathname element-type [Function]
The map-file-to-ivector function tries to open the file at pathname for reading. If successful, the function maps the file's contents to a range of virtual addresses. If successful, it returns a read-only vector whose element-type is given by element-type, and whose contents are the contents of the memory-mapped file.

pathname
The pathname of the file to be memory-mapped.

element-type
The element-type of the vector to be created. Specified as a type-specifier that names a subtype of either signed-byte or unsigned-byte.

The returned vector is a displaced-array whose element-type is (UPGRADED-ARRAY-ELEMENT-TYPE element-type). The target of the displaced array is a vector of type (SIMPLE-ARRAY element-type (*)) whose elements are the contents of the memory-mapped file.

Because of alignment issues, the mapped file's contents start a few bytes (4 bytes on 32-bit platforms, 8 bytes on 64-bit platforms) into the vector. The displaced array returned by map-file-to-ivector hides this overhead, but it's usually more efficient to operate on the underlying simple 1-dimensional array. Given a displaced array (like the value returned by map-file-to-ivector), the function array-displacement returns the underlying array and the displacement index in elements.

Currently, Clozure CL supports only read operations on memory-mapped files. If you try to change the contents of an array returned by map-file-to-ivector, Clozure CL signals a memory error.

unmap-ivector displaced-array [Function]
If the argument is a displaced-array returned by map-file-to-ivector, and if it has not yet been unmapped by this function, then unmap-ivector undoes the memory mapping, closes the mapped file, and changes the displaced-array so that its target is an empty vector (of length zero).

map-file-to-octet-vector displaced-array [Function]
This function is a synonym for (map-file-to-ivector pathname '(unsigned-byte 8)) It is provided as a convenience for the common case of memory-mapping a file as a vector of bytes.

unmap-octet-vector displaced-array [Function]
This function is a synonym for unmap-ivector

Static Variables

Clozure CL supports the definition of static variables, whose values are the same across threads, and which may not be dynamically bound. The value of a static variable is thus the same across all threads; changing the value in one thread changes it for all threads.

Attempting to dynamically rebind a static variable (for instance, by using LET, or using the variable name as a parameter in a LAMBDA form) signals an error. Static variables are shared global resources; a dynamic binding is private to a single thread.

Static variables therefore provide a simple way to share mutable state across threads. They also provide a simple way to introduce race conditions and obscure bugs into your code, since every thread reads and writes the same instance of a given static variable. You must take care, therefore, in how you change the values of static variables, and use normal multithreaded programming techniques, such as locks or semaphores, to protect against race conditions.

In Clozure CL, access to a static variable is usually faster than access to a special variable that has not been declared static.

defstatic var value &key docstring [Macro]
Proclaims the variable special, assigns the variable the supplied value, and assigns the docstring to the variable's variable documentation. Marks the variable static, preventing any attempt to dynamically rebind it. Any attempt to dynamically rebind var signals an error.

Saving Applications

Clozure CL provides the function save-application, which creates a file containing an archived Lisp memory image.

Clozure CL consists of a small executable called the lisp kernel, which implements the very lowest level features of the Lisp system, and a heap image, which contains the in-memory representation of most of the Lisp system, including functions, data structures, variables, and so on. When you start Clozure CL, you are launching the kernel, which then locates and reads an image file, restoring the archived image in memory. Once the image is fully restored, the Lisp system is running.

Using save-application, you can create a file that contains a modified image, one that includes any changes you've made to the running Lisp system. If you later pass your image file to the Clozure CL kernel as a command-line parameter, it then loads your image file instead of its default one, and Clozure CL starts up with your modifications.

If this scenario seems to you like a convenient way to create an application, that's just as intended. You can create an application by modifying the running Lisp until it does what you want, then use save-application to preserve your changes and later load them for use.

In fact, you can go further than that. You can replace Clozure CL's toplevel function with your own, and then, when the image is loaded, the Lisp system immediately performs your tasks rather than the default tasks that make it a Lisp development system. If you save an image in which you have done this, the resulting Lisp system is your tool rather than a Lisp development system.

You can go a step further still. You can tell save-application to prepend the Lisp kernel to the image file. Doing this makes the resulting image into a self-contained executable binary. When you run the resulting file, the Lisp kernel immediately loads the attached image file and runs your saved system. The Lisp system that starts up can have any behavior you choose. It can be a Lisp development system, but with your customizations; or it can immediately perform some task of your design, making it a specialized tool rather than a general development system.

In other words, you can develop any application you like by interactively modifying Clozure CL until it does what you want, then using save-application to preserve your changes in an executable image.

On Mac OS X, the application builder uses save-application to create the executable portion of the application bundle. Double-clicking the application bundle runs the executable image created by save-application.

Also on Mac OS X, Clozure CL supports an object type called macptr, which is the type of pointers into the foreign (Mac OS) heap. Examples of commonly-user macptr objects are Cocoa windows and other dynamically-allocated Mac OS system objects.

Because a macptr object is a pointer into a foreign heap that exists for the lifetime of the running Lisp process, and because a saved image is used by loading it into a brand new Lisp process, saved macptr objects cannot be relied on to point to the same things when reconstituted from a saved image. In fact, a restored macptr object might point to anything at all-for example an arbitrary location in the middle of a block of code, or a completely nonexistent virtual address.

For that reason, save-application converts all macptr objects to dead-macptr objects when writing them to an image file. A dead-macptr is functionally identical to a macptr, except that code that operates on macptr objects distinguishes them from dead-macptr objects and can handle them appropriately-signaling errors, for example.

Tthere is one exception to the conversion of macptr to dead-macptr objects: a macptr object that points to the address 0 is not converted, because address 0 can always be relied upon to refer to the same thing.

The constant +null-ptr+ refers to a macptr object that points to address 0.

On all supported platforms, you can use save-application to create a command-line tool that runs very like any other command-line program does. Alternatively, if you choose not to prepend the kernel, you can save an image and then later run it by passing it as a command-line parameter to the ccl or ccl64 script.

save-application filename &key toplevel-function init-file error-handler application-class clear-clos-caches (purify t) impurify (mode #o644) prepend-kernel native [Function]
Saves a heap image.

filename
The pathname of the file to be created when Clozure CL saves the application.

toplevel-function
The function to be executed after startup is complete. The toplevel is a function of no arguments that performs whatever actions the lisp system should perform when launched with this image.

If this parameter is not supplied, Clozure CL uses its default toplevel. The default toplevel runs the read-eval-print loop.

init-file
The pathname of a Lisp file to be loaded when the image starts up. You can place initialization expressions in this file, and use it to customize the behavior of the Lisp system when it starts up.

error-handler
The error-handling mode for the saved image. The supplied value determines what happens when an error is not handled by the saved image. Valid values are :quit (Lisp exits with an error message); :quit-quietly (Lisp exits without an error message); or :listener (Lisp enters a break loop, enabling you to debug the problem by interacting in a listener). If you don't supply this parameter, the saved image uses the default error handler (:listener).

application-class
The CLOS class that represents the saved Lisp application. Normally you don't need to supply this parameter; save-application uses the class ccl:lisp-development-system. In some cases you may choose to create a custom application class; in that case, pass the name of the class as the value for this parameter.

clear-clos-caches
If true, ensures that CLOS caches are emptied before saving the image. Normally you don't need to supply this parameter, but if for some reason you want to ensure the CLOS caches are clear when the image starts up, you can pass any true value.

purify
When true, calls (in effect) purify before saving the heap image. This moves certain objects that are unlikely to become garbage to a special memory area that is not scanned by the GC (since it is expected that the GC wouldn't find anything to collect).

impurify
If true, calls (in effect) impurify before saving the heap image. (If both :impurify and :purify are true, first impurify is done, and then purify.)

impurify moves objects in certain special memory areas into the regular dynamic heap, where they will be scanned by the GC.

mode
A number specifying the mode (permission bits) of the output file.

prepend-kernel
Specifies the file to prepend to the saved heap image. A value of t means to prepend the lisp kernel binary that the lisp started with. Otherwise, the value of :prepend-kernel should be a pathname designator for the file to be prepended.

If the prepended file is execuatable, its execute mode bits will be copied to the output file.

This argument can be used to prepend any kind of file to the saved heap image. This can be useful in some special cases.

native
If true, saves the image as a native (ELF, Mach-O, PE) shared library. (On platforms where this isn't yet supported, a warning is issued and the option is ignored.)

*save-exit-functions* [Variable]
This variable contains a list of 0-argument functions that will be called before saving a heap image. Users may add functions to this list as needed.

*restore-lisp-functions* [Variable]
This variable contains a list of 0-argument functions that will be called after restoring a saved heap image. Users may add functions to this list as needed.

*lisp-cleanup-functions* [Variable]
This variable contains a list of 0-argument functions that will be called before quitting the lisp.

Note that save-application quits the lisp, so any functions on this list will be invoked before saving a heap image.

*lisp-startup-functions* [Variable]
This variable contains a list of 0-argument functions that will be called after starting the lisp.

Concatenating FASL Files

Multiple fasl files can be concatenated into a single file. The single file might be easier to distribute or install, and loading it may be slightly faster than loading the individual files (since it avoids the overhead of opening and closing each file in succession).

fasl-concatenate output-file fasl-files &key (:if-exists :error) [Function]
This function reads the fasl files specified by the list fasl-files and combines them into a single fasl file named output-file. The :if-exists keyword argument is interpreted as in the standard function open.

Loading the concatenated fasl file has the same effect as loading the invidual input fasl files in the specified order.

The pathname-type of the output file and of each input file defaults to the current platform's fasl file type (see Platform-specific filename conventions). If any of the input files has a different type an error will be signaled, but fasl-concatenate doesn't otherwise try too hard to verify that the input files are real fasl files for the current platform.

Floating Point Numbers

In Clozure CL, the Common Lisp types short-float and single-float are implemented as IEEE single precision values; double-float and long-float are IEEE double precision values. On 64-bit platforms, single-floats are immediate values (like fixnums and characters).

Floating-point exceptions are generally enabled and detected. By default, threads start up with overflow, division-by-zero, and invalid enabled, and the rounding mode is set to nearest. The functions set-fpu-mode and get-fpu-mode provide user control over floating-point behavior.

get-fpu-mode &optional mode [Function]
Return the state of exception-enable and rounding-mode control flags for the current thread.

When called without the optional mode argument, this function returns a plist of keyword/value pairs which describe the floating point exception-enable and rounding-mode flags for the current thread.

If mode is supplied, it should be one of the keywords :rounding-mode, :overflow, :underflow, :division-by-zero, :invalid, or :inexact. The value of corresponding control flag is returned.

rounding-mode
One of :nearest, :zero, :positive, or :negative

overflow, underflow, division-by-zero, invalid, inexact
If t, the floating-point exception is signaled. If nil, it is masked.

set-fpu-mode &key rounding-mode overflow underflow division-by-zero invalid inexact [Function]
Set the state of exception-enable and rounding-mode control flags for the current thread.

rounding-mode
If supplied, must be one of :nearest, :zero, :positive, or :negative.

overflow, underflow, division-by-zero, invalid, inexact
NIL to mask the exception, T to signal it.

Sets the current thread's exception-enable and rounding-mode control flags to the indicated values for arguments that are supplied, and preserves the values assoicated with those that aren't supplied.

Code Coverage

Overview

In Clozure CL 1.4 and later, code coverage provides information about which paths through generated code have been executed and which haven't. For each source form, it can report one of three possible outcomes:

Not covered: this form was never entered.

Partly covered: This form was entered, and some parts were executed and some weren't.

Fully covered: Every bit of code generated from this form was executed.

Limitations

While the information gathered for coverage of generated code is complete and precise, the mapping back to source forms is of necessity heuristic, and depends a great deal on the behavior of macros and the path of the source forms through compiler transforms. Source information is not recorded for variables, which further limits the source mapping. In practice, there is often enough information scattered about a partially covered function to figure out which logical path through the code was taken and which wasn't. If that doesn't work, you can try disassembling to see which parts of the compiled code were not executed: in the disassembled code there will be references to #<CODE-NOTE [xxx] ...> where xxx is NIL if the code that follows was never executed and non-NIL if it was.

Sometimes the situation can be improved by modifying macros to try to preserve more of the input forms, rather than destructuring and rebuilding them.

Because the code coverage information is associated with compiled functions, code coverage information is not available for load-time toplevel expressions. You can work around this by creating a function and calling it. I.e. instead of

(progn
  (do-this)
  (setq that ...) ...))
do:
(defun init-this-and-that ()
  (do-this)
  (setq that ...)  ...)
(init-this-and-that)
Then you can see the coverage information in the definition of init-this-and-that.

Usage

In order to gather code coverage information, you first have to recompile all your code to include code coverage instrumentation. Compiling files will generate code coverage instrumentation if ccl:*compile-code-coverage* is true:

(setq ccl:*compile-code-coverage* t) 
(recompile-all-your-files)
The compilation process will be many times slower than normal, and the fasl files will be many times bigger.

When you execute functions loaded from instrumented fasl files, they will record coverage information every time they are executed. You can examine that information by calling ccl:report-coverage or ccl:coverage-statistics.

While recording coverage, you can collect incremental coverage deltas between any two points in time. You might do this while running a test suite, to record the coverage for each test, for example:

(ccl:reset-incremental-coverage)
(loop with coverage = (make-hash-table)
      for test in (tests-to-run)
      do (run-test test)
      do (setf (gethash test coverage) (ccl:get-incremental-coverage))
      finally (return coverage))
creates a hash table mapping a test to a representation of all coverage recorded while running the test. This hash table can then be passed to ccl:report-coverage, ccl:incremental-coverage-svn-matches or ccl:incremental-coverage-source-matches.
Functions and Variables

The following functions can be used to manage the coverage data:

report-coverage output-file &key (tags nil) (external-format :default) (statistics t) (html t) [Function]
Generate a code coverage report

output-file
Pathname for the output index file.

html
If non-nil (the default), this will generate an HTML report, consisting of an index file in output-file and, in the same directory, one html file for each instrumented source file that has been loaded in the current session.

tags
If non-nil, this should be a hash table mapping arbitrary keys (tags) to incremental coverage deltas. The HTML report will show a list of tags, and allow selection of an arbitrary subset of them to show the coloring and statistics for coverage by that subset.

external-format
Controls the external format of the html files.

statistics
If non-nil (the default), a comma-separated file is generated with the summary of statistics. You can specify a filename for the statistics argument, otherwise statistics.csv is created in the directory of output-file. See documentation of coverage-statistics below for a description of the values in the statistics file.

If you've loaded foo.lx64fsl and bar.lx64fsl, and have run some tests, you could do

(report-coverage "/my/dir/coverage/report.html")
and this would generate report.html, foo_lisp.html and bar_lisp.html, and statistics.csv all in /my/dir/coverage/.
reset-coverage [Function]
Resets all coverage data back to the “Not Executed” state

Resets all coverage data back to the “Not Executed” state

clear-coverage [Function]
Forget about all instrumented files that have been loaded.

Gets rid of the information about which instrumented files have been loaded, so ccl:report-coverage will not report any files, and ccl:save-coverage-in-file will not save any info, until more instrumented files are loaded.

save-coverage-in-file pathname [Function]
Save all coverage into to a file so you can restore it later.

Saves all coverage info in a file, so you can restore the coverage state later. This allows you to combine multiple runs or continue in a later session. Equivalent to (ccl:write-coverage-to-file (ccl:get-coverage) pathname).

restore-coverage-from-file pathname [Function]
Load coverage state from a file.

Restores the coverage data previously saved with ccl:save-coverage-in-file, for the set of instrumented fasls that were loaded both at save and restore time. I.e. coverage info is only restored for files that have been loaded in this session. For example if in a previous session you had loaded "foo.lx86fsl" and then saved the coverage info, in this session you must load the same "foo.lx86fsl" before calling restore-coverage-from-file in order to retrieve the stored coverage info for "foo". Equivalent to (ccl:restore-coverage (ccl:read-coverage-from-file pathname)).

get-coverage [Function]
Returns a snapshot of the current coverage data.

Returns a snapshot of the current coverage data. A snapshot is a copy of the current coverage state. It can be saved in a file with ccl:write-coverage-to-file, reinstated back as the current state with ccl:restore-coverage, or combined with other snapshots with ccl:combine-coverage.

restore-coverage snapshot [Function]
Reinstalls a coverage snapshot as the current coverage state.

Reinstalls a coverage snapshot as the current coverage state.

combine-coverage snapshots [Function]
Combines multiple coverage snapshots into one.

Takes a list of coverage snapshots and returns a new coverage snapshot representing a union of all the coverage data.

write-coverage-to-file snapshot pathname [Function]
Save a coverage snapshot in a file.

Saves the coverage snapshot in a file. The snapshot can be loaded back with ccl:read-coverage-from-file or loaded and restored with ccl:restore-coverage-from-file. Note that the file created is actually a lisp source file and can be compiled for faster loading.

read-coverage-from-file pathname [Function]
Return the coverage snapshot saved in a file.

Returns the snapshot saved in pathname. Doesn't affect the current coverage state. pathname can be the file previously created with ccl:write-coverage-to-file or ccl:save-coverage-in-file, or it can be the name of the fasl created from compiling such a file.

coverage-statistics [Function]
Returns a sequence of ccl:coverage-statistics objects, one per source file.

Returns a sequence of ccl:coverage-statistics objects, one for each source file, containing the same information as that written to the statistics file by report-coverage. The following accessors are defined for ccl:coverage-statistics objects:

coverage-source-file
the name of the source file corresponding to this information

coverage-expressions-total
the total number of expressions

coverage-expressions-entered
the number of source expressions that have been entered (i.e. at least partially covered)

coverage-expressions-covered
the number of source expressions that were fully covered

coverage-unreached-branches
the number of conditionals with one branch taken and one not taken

coverage-code-forms-total
the total number of code forms. A code form is an expression in the final stage of compilation, after all macroexpansion and compiler transforms and simplification

coverage-code-forms-covered
the number of code forms that have been entered

coverage-functions-total
the total number of functions

coverage-functions-fully-covered
the number of functions that were fully covered

coverage-functions-partly-covered
the number of functions that were partly covered

coverage-functions-not-entered
the number of functions never entered

reset-incremental-coverage [Function]
Reset incremental coverage.

Marks a starting point for recording incremental coverage. Note that calling this function does not affect regular coverage data (whereas calling ccl:reset-coverage resets incremental coverage as well).

get-incremental-coverage &key (reset t) [Function]
Returns the delta of coverage since the last incremental reset.

Returns the delta of coverage since the last reset of incremental coverage. If reset is true (the default), it also resets incremental coverage now, so that the next call to get-incremental-coverage will return the delta from this point.

Incremental coverage deltas are represented differently than the full coverage snapshots returned by functions such as ccl:get-coverage. Incremental coverage uses an abbreviated format and is missing some of the information in a full snapshot, and therefore cannot be passed to functions documented to accept a snapshot, only to functions specifically documented to accept incremental coverage deltas.

incremental-coverage-source-matches collection sources [Function]
Find incremental coverage deltas intersecting source regions.

collection
A hash table mapping arbitrary keys to incremental coverage deltas, or a sequence of incremental coverage deltas.

sources
A list of pathnames and/or source-notes, the latter representing a range within a file.

Given a hash table collection whose values are incremental coverage deltas, return a list of all keys corresponding to those deltas that intersect any region in sources.

For example if the deltas represent tests, then the returned value is a list of all tests that cover some part of the source regions.

collection can also be a sequence of deltas, in which case a subsequence of matching deltas is returned. In particular you can test whether any particular delta intersects the sources by passing it in as a single-element list.

incremental-coverage-svn-matches collection &key (directory (current-directory)) (revision :base) [Function]
Find incremental coverage deltas matching changes from a particular subversion revision.

collection
A hash table mapping arbitrary keys to incremental coverage deltas, or a sequence of incremental coverage deltas.

directory
The pathname of a subversion working directory.

revision
The revision to compare to the working directory, an integer or another value whose printed representation is suitable for passing as the --revision argument to svn.

Given a hash table collection whose values are incremental coverage deltas, return a list of all keys corresponding to those deltas that intersect any changed source in directory since revision revision in subversion.

For example if the deltas represent tests, then the returned value is a list of all tests that might be affected by the changes.

collection can also be a sequence of deltas, in which case a subsequence of matching deltas is returned. In particular you can test whether any particular delta is affected by the changes by passing it in as a single-element list.

*compile-code-coverage* [Variable]
When true, instrument functions being compiled to collect code coverage information.

This variable controls whether functions are instrumented for code coverage. Files compiled while this variable is true will contain code coverage instrumentation.

without-compiling-code-coverage [Macro]
Don't record code coverage for forms within the body.

This macro arranges so that body doesn't record internal details of code coverage. It will be considered totally covered if it's entered at all. The Common Lisp macros ASSERT and CHECK-TYPE use this macro.

Interpreting Code Coloring

The output of ccl:report-coverage consists of formatted source code, with coverage indicated by coloring. Four colors are used: dark green for forms that compiled to code in which every single instruction was executed, light green for forms that have been entered but weren't totally covered, red for forms that were never entered, and the page background color for toplevel forms that weren't instrumented.

The source coloring is applied from outside in. So for example if you have

(outer-form ... (inner-form ...) ...)
  
first the whole outer form is painted with whatever color expresses the outer form coverage, and then the inner form color is replaced with whatever color expresses the inner form coverage. One consequence of this approach is that every part of the outer form that is not specifically inside some executable inner form will have the outer form's coverage color. If the syntax of outer form involves some non-executable forms, or forms that do not have coverage info of their own for whatever reason, then they will just inherit the color of the outer form, because they don't get repainted with a color of their own.

One case in which this approach can be confusing is in the case of symbols. As noted in the Limitations section, coverage information is not recorded for variables; hence the coloring of a variable does not convey information about whether the variable was evaluated or not -- that information is not available, and the variable just inherits the color of the form that contains it.

Other Extensions

unwind-protect protected-form {cleanup-form}* [Macro]
Ensure cleanup-forms are executed.

In Clozure CL, the cleanup forms are always executed as if they were wrapped with without-interrupts. To allow interrupts, use with-interrupts-enabled.


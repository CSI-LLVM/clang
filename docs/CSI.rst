Comprehensive Static Instrumentation
====================================

Introduction
------------

CSI:LLVM is a framework providing comprehensive static instrumentation via the
compiler in order to simplify the task of building efficient and effective
platform-independent dynamic-analysis tools.  The CSI:LLVM compiler pass inserts
instrumentation hooks at salient locations throughout the compiled code of a
program-under-test, such as function entry and exit points, basic-block entry
and exit point, before and after each memory operation, etc.  Tool writers can
instrument a program-under-test simply by first writing a library that defines
the relevant hooks and statically linking their compiled library with the
program-under-test.

Supported Platforms
-------------------

CSI is currently only supported on Linux x86_64 (tested on UBuntu 14.04 x86_64).
To ensure high performance of CSI tools, CSI:LLVM ideally should be configured
to enable link-time optimization (LTO), and the GNU ``gold linker`` is a
prerequisite for enabling LTO for CSI:LLVM.  (See
`http://llvm.org/docs/LinkTimeOptimization.html` for more detail on LLVM LTO.)

Usage: Create a CSI tool
------------------------

To create a CSI tool, add ``#include <csi.h>`` at the top of the tool source
and implement function bodies for the hooks relevant to the tool.

To build the tool object file suitable for linking with an instrumented
program-under-test (assuming the tool source file is named ``my-tool.cpp``),
execute the following:

.. code-block:: bash

  % clang++ -c -emit-llvm null-tool.c -o null-tool.o
  % clang++ -c -emit-llvm my-tool.cpp -o my-tool.o
  % llvm-link my-tool.o null-tool.o -o my-tool.o

The ``null-tool.c`` file is provided as part of the CSI distribution (under
``llvm/projects/compiler-rt/test/csi/tools/null-tool.c``) which consists
of null hooks that simply return.  Linking ``my-tool`` with ``null-tool``
allows the LTO to later elide hooks irrelevant to the tool entirely from the
program-under-test.

The LLVM/Clang used to build the tool does not have to be CSI:LLVM, as long
as it generates LLVM bitcode compatible with CSI:LLVM.

Usage: Create a CSI instrumented program-under-test
---------------------------------------------------

To create a CSI instrumented program-under-test linked with a CSI tool
(henceforth referred to as the Tool-Instrumented-Executable, or TIX for short),
one needs to do the following:

* Modify paths in the build process to point to CSI:LLVM (including its Clang
  driver).
* When building object files for the TIX, pass additional arguments ``-fcsi``
  and ``-emit-llvm`` to the Clang driver, which produces CSI instrumented
  object files.
* During the linking stage for the TIX, add additional arguments
  ``-fuse-ld=gold`` and ``-flto`` and add the tool object file (e.g.
  ``my-tool.o``) to be statically linked to the TIX.

For example, say we want to instrument a program that consists of two files
``foo.cpp`` and ``bar.cpp`` and link the program with a CSI tool ``my-tool.o``
(built as shown above), execute the following:

.. code-block:: bash

  % clang++ -c -O3 -g -fcsi -emit-llvm foo.cpp -o foo.o
  % clang++ -c -O3 -g -fcsi -emit-llvm bar.cpp -o bar.o
  % clang++ foo.o bar.o my-tool.o libclang_rt.csi-x86_64.a -fuse-ld=gold -flto -lrt -ldl -o foo

Notice that in the final stage of linking, the tool user also needs to link in
the static library of the CSI runtime to produce the final TIX.  The runtime
archive is distributed under the ``build/lib/clang/<VERSION>/lib/<OS>``
directory. We plan to investigate means of linking with the runtime
automatically in the future, but for the time being, the tool user should link
it in explicitly.

CSI API Overview
----------------

CSI's instrumentation hooks are organized into four groups: initialization,
memory accesses, basic blocks, and functions.  To provide flexibility to the
tool writer, a hook exists both just before the event, and just after.

Except for the initialization hooks, every other hook names one or more
program objects, such as a basic block or a memory operation.  CSI gives
each such program object a unique integer identifier within one of
(currently) six program-object categories:

* functions,
* function exits,
* basic blocks,
* call sites,
* loads, and
* stores.

Within each category, the ID's are consecutively numbered from 0 up
to the number of such objects minus 1.  The range of ID's for each
category is extended during unit initialization, which happens at the
beginning of the program.  In the case of dynamic loading, it will
also occur as new units are loaded in.  By maintaining a contiguous
set of ID's, the tool writer can easily track program objects and iterate
through all objects in a category.

To relate a given program object to locations in the source code, CSI
provides also front-end data (FED) tables, which provide file name and
source lines for each program object given the object's ID.

CSI API: Initialization Hooks
-----------------------------

CSI provides two initialization hooks, shown below:

.. code-block:: c++

  typedef int64_t csi_id_t;

  // Value representing unknown CSI ID
  #define UNKNOWN_CSI_ID ((csi_id_t)-1)

  typedef struct {
    csi_id_t num_bb;
    csi_id_t num_callsite;
    csi_id_t num_func;
    csi_id_t num_func_exit;
    csi_id_t num_load;
    csi_id_t num_store;
  } instrumentation_counts_t;

  // Hooks to be defined by tool writer
  void __csi_init();
  void __csi_unit_init(const char * const file_name, const instrumentation_counts_t counts);

Instrumentation hook ``__csi_init`` is designed for performing any
global initialization necessary for the tool; it is called once only
when the instrumented program loads, before both the execution of the
``main`` function and the initialization of global variables.  The
``__csi_init`` hook is assigned with the highest execution priority and is
typically called before any other constructor.  If the program-under-test also
contains a constructor annotated with the highest priority (via the
``init_priority`` attribute), however, the execution order of that constructor
relative to ``__csi_init`` is undefined.

In addition to the global initialization hook, CSI also provides the
translation-unit initialization hook ``__csi_unit_init``, called once when a
translation unit --- a source file, an object file, or a bitcode file --- loads.
The ``file_name`` parameter provides the name of the source file corresponding
to the translation unit.  The hook provides parameters for the number of each
instrumentation type in the unit.  This allows a tool to prepare any data
structures ahead of time.

When multiple translation units contribute to the TIX, the tool writer may not
assume that the invocations of ``__csi_unit_init`` are called in any particular
order, except that they all occur before ``main``.  In the case of a
dynamic library compiled with CSI, ``__csi_unit_init`` is invoked once per
translation unit that contributes to the dynamic library at the time that the
library loads.


CSI API: Functions
------------------

CSI provides hooks for function entry and exit, shown below:

.. code-block:: c++

  void __csi_func_entry(const csi_id_t func_id);
  void __csi_func_exit(const csi_id_t func_exit_id, const csi_id_t func_id);

The hook ``__csi_func_entry`` is invoked at the beginning of every
instrumented function instance after the function has been entered and
initialized but before any user code has run.  The ``func_id`` parameter
identifies the function being entered or exited.  Correspondingly, the
hook ``__csi_func_exit`` is invoked just before the function returns
normally).  (We have not yet defined the API for exceptions.)
The ``func_exit_id`` parameter allows the tool writer to distinguish the
potentially multiple function exits, and the ``func_id`` ID identifies
the function that the hook is in.

CSI API: Basic Blocks
---------------------

CSI also provide instrumentation hooks basic block entry and exit.
A basic block consists of strands of instructions with no incoming branches
except to its entry point, and no outgoing branches except from its exit point.
The API hooks for basic blocks are shown below:

.. code-block:: c++

 void __csi_bb_entry(const csi_id_t bb_id);
 void __csi_bb_exit(const csi_id_t bb_id);

The hook ``__csi_bb_entry`` is called when control enters a basic block,
and ``__csi_bb_exit`` is called just before control leaves the basic
block.  The ``bb_id`` parameter identifies the entered or exited basic
block.  The ``__csi_func_entry/exit`` and ``__csi_bb_entry/exit`` are
properly nested: before entering the first basic block in a function,
``__csi_func_entry`` is invoked before ``__csi_bb_entry``; before
returning from a function, ``__csi_bb_exit`` is invoked before
``__csi_func_exit``.


CSI API: Function Calls
-----------------------

CSI provides the following hooks for call sites:

.. code-block:: c++

  void __csi_before_call(const csi_id_t call_id, const csi_id_t func_id);
  void __csi_after_call(const csi_id_t call_id, const csi_id_t func_id);

The ``call_id`` parameter identifies the call site, and the ``func_id``
parameter identifies the called function.  Note that it may not always be
possible to CSI to produce the function ID corresponds to the called function
statically --- for example, if a function is called indirectly
through a function pointer or if the function called is an uninstrumented
function.  In such scenarios, the value of the ``func_id`` will be
``UNKNOWN``, a macro defined to have type ``csi_id_t`` with value ``-1``.

CSI API: Memory Operations
--------------------------

CSI provides the following hooks for memory operations:

.. code-block:: c++

  void __csi_before_load(const csi_id_t load_id, const void *addr,
                         const int32_t num_bytes, const uint64_t prop);
  void __csi_after_load(const csi_id_t load_id, const void *addr,
                        const int32_t num_bytes, const uint64_t prop);
  void __csi_before_store(const csi_id_t store_id, const void *addr,
                          const int32_t num_bytes, const uint64_t prop);
  void __csi_after_store(const csi_id_t store_id, const void *addr,
                         const int32_t num_bytes, const uint64_t prop);

  // Load property: the load is a read-before-write on the address in
  // the same basic block.
  #define CSI_PROP_LOAD_READ_BEFORE_WRITE_IN_BB 0x1

The hooks ``__csi_before_load`` and ``__csi_after_load`` are called before and
after memory loads, respectively, and likewise, ``__csi_before_store`` and
``__csi_after_store`` are called before and after memory stores.  The parameter
``addr`` is the address of the memory accessed, and ``num_bytes`` is the number
of bytes loaded or stored.  The ``prop`` parameter is a property: a 64-bit
unsigned integer that CSI uses to export the results of compiler analysis and
other information known at compile time.  A particular property of the memory
operation is encoded as a bit field in ``prop``, which can be checked against
the property macros defined by CSI.  Currently, the only property implemented is
whether a load is a read-before-write within the basic block enclosing it.  We
plan to extend the CSI to include more property values and incorporate property
into other types of hooks.

CSI API: Front-End Data (FED) Tables
------------------------------------

CSI provides a front-end data (FED) table for each type of
program objects to allow a tool to easily relate runtime events back to
locations in the source code.  The FED tables are indexed by the program
object's ID.  The accessors for the FED tables are shown below:

.. code-block:: c++

  typedef struct {
    char * filename;
    int32_t line_number;
  } source_loc_t;

  // Accessors for various CSI FED tables.
  // Return NULL when given an invalid ID.
  source_loc_t const * __csi_get_func_source_loc(const csi_id_t func_id);
  source_loc_t const * __csi_get_func_exit_source_loc(const csi_id_t func_exit_id);
  source_loc_t const * __csi_get_bb_source_loc(const csi_id_t bb_id);
  source_loc_t const * __csi_get_call_source_loc(const csi_id_t call_id);
  source_loc_t const * __csi_get_load_source_loc(const csi_id_t load_id);
  source_loc_t const * __csi_get_store_source_loc(const csi_id_t store_id);

We describe the interface of the accessors for the basic-block FED table, and
accessors for the other FED tables work similarly.  Given a ``bb_id``
corresponding to a basic block, as the parameter passed into the hooks for the basic
block entry and exit, ``__csi_get_bb_source_loc`` returns a ``struct`` that
contains the source location of the basic block, including the filename of the
translation unit that the basic block belongs to and its begin (inclusive) line
numbers.  The type for the line number is signed, which permits an error value of
``-1`` for when the line-number information is not available.

Currently the FED tables are initialized by default, which incurs some runtime
overhead.  We are considering providing explicit initialization calls for the
FED tables in the future as an optimization, which allows the runtime to
optimize away the cost of FED table initialization unless the tool explicitly
request a particular FED table to be initialized.


Limitations
-----------

* One limitation to LTO is that, it cannot fully optimize dynamic libraries,
  since dynamic libraries must be compiled as position independent code (PIC),
  and as the compiler cannot predict runtime addresses within the library,
  it must invoke tool-provided hooks as PIC function calls.  In these cases,
  LTO can sometimes fail to perform optimization to eliminate null hooks or
  dead code within the hooks.  To be conservative and avoid these penalties,
  libraries should be statically linked with the TIX.

* On systems where LTO is not used, the TIX produced by linking a program with
  a CSI tool will still function correctly, but might not be optimized.  Null
  hooks might not be elided, for example, meaning that linking an instrumented
  program-under-test with the null tool might produce a slower executable than
  if CSI instrumentation were not inserted.

* CSI currently does not support instrumentation for exceptions and C++11 atomics.


Current Status
--------------

This is the first release of CSI.  It has been tested with large C++ programs,
such as the Apache HTTP server (version 2.4.17), but we don't promise that it's
bug free.

We are actively working on enhancing the CSI framework, and we have a few minor
milestones and major milestones planned.  The minor milestones that we are
actively developing include the following:

* Incorporate more properties to expose additional compiler analyses and other
  information known at compile time, such as whether a memory access is a
  constant, whether a variable accessed is captured, and such.

* Extend properties to other types of hooks.

* Incorporate more detailed information into the FED tables.  Specifically, the
  return type ``source_loc_t`` struct currently contains only the begin source
  line number.  We plan to include also the end (exclusive) line number, the begin
  and end column numbers.

The major milestones that we are considering include:

* Add instrumentation for exceptions.

* Add instrumentation for C++11 atomics.

* Providing additional static information such as how the program objects relate to
  each other.

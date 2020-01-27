Cross-compilation with GHC
==========================

Wouldn't it be awesome if GHC was able to cross-compile Haskell code to
WebAssembly, JavaScript, Java bytecode, etc.? Sadly it can't be done directly
with stock GHC. Several glorious attempts at doing cross-compilation (e.g.
GHCJS, Asterius) are based on GHC forks and rely on less glorious hacks to work.

This document explains why cross-compiling Haskell code with GHC is so
difficult. It also describes what would need to be done to make GHC an effective
cross-compiler.

If there is anything unclear or wrong, please open an issue, send a pull request
or send me an email (sylvain@haskus.fr).

Confined mode
-------------

Historically GHC has been designed with the hypothesis the code it produces is
to be run on the same platform as the compiler itself. Even more than that, the
code objects it produces (`.o`, `.a`, `.so`, `.dll`, etc.) must be compatible
with the code objects used to build the compiler itself: boot libraries
(`bytestring`, `base` and all the others in the `libraries` directory in GHC's
source tree) and the compiler itself if the produced code uses the GHC API.

To refer to this mode I will use the expression **confined mode**.

In confined mode everything is simple and easy:

* we want compiler plugins: just dynamically link modules containing plugins
  with the compiler. By hypothesis they are compatible with the compiler and the
  boot libraries.

* we want to have an interactive interpreter (ghci): build the code (either as
  ByteCode or as normal code objects), load it like a plugin above and execute
  it.

* but the code uses external libraries (libgmp, libwhatever)! No problem, we
  can dynamically link them too.

* we need to interpret some code at compilation time (Template Haskell): just
  reuse the interpreter and it's done.

It's obviously not that easy to implement but conceptually it is. Sadly most of
this breaks when we try to get out of the confined mode.

Breach #1: stage 1 compiler
---------------------------

If confined mode was strictly enforced, GHC could only be executed on the
platform it was initially developed on. It's obviously not the case and it is
the first breach in the confined mode: GHC stages.

If we disable GHC features that require code execution (plugins and the internal
interpreter, hence Template Haskell too), we get a GHC that can produce code for
a different target. The price to pay is that this compiler doesn't support the
full language.

GHC itself is written with this language subset (it doesn't use compiler plugins
nor Template Haskell).

A GHC compiler trimmed of these features is called **stage 1** compiler. **stage
0** is the GHC compiler used to build stage 1. **stage 2** is the full compiler
built by stage 1 (potentially on a different platform than stage 0 host).


Breach #2: RTS ways
-------------------

GHC can produce different code objects depending on some options. For example,
it can produce objects that:

- use the multi-threaded runtime system or not
- support profiling or not
- dump additional debug information or not
- use different heap object representation (e.g. `tables_next_to_code`)
- support dynamic linking or not

These options are called "compiler ways". Some of them can be combined (e.g.
threaded + debugging).

Depending on the selected way, the compiler produces and links appropriate
objects together. These objects are identified by a suffix: e.g. `*.p_o` for an
object built with profiling enabled; `*.thr_debug_p.a` for an archive built with
multi-threading, debugging, and profiling enabled. See the gory details on the
`wiki <https://gitlab.haskell.org/ghc/ghc/wikis/commentary/rts/compiler-ways>`_.

Installed packages usually don't provide objects for all the possible ways as it
would make compilation times and disk space explode for features rarely used.

If the selected way is not the same as the one used to build GHC and its boot
libraries, it breaks the confined mode assumptions: the produced objects can't
be dynamically linked with GHC.

GHC can build objects both for itself (i.e. using the way it has been built
with) and for the selected target way. By doing this, it can simulate the
confined mode by loading objects on the host that are different from the objects
produced for the target, with the hope that the two ways have no observable
difference. Quoting the `wiki
<https://gitlab.haskell.org/ghc/ghc/wikis/remote-GHCi>`_: "The way this is done
currently is inherently unsafe, because we use the profiled .hi files with the
unprofiled object files, and hope that the two are in sync."

Another solution is to use an external interpreter.


Breach #3: external interpreter
-------------------------------

The idea behind the external interpreter is to delegate the execution of the
target code to another process (called `iserv`). This process can then delegate
to another one hosted on another platform or in a VM (e.g. NodeJS), etc.. 

GHC performs two-way communication with this process to send ByteCode to
evaluate, to ask for package to be linked, etc. During code execution, the
`iserv` process may query the host GHC (e.g. when Template Haskell code is run,
it may query information about some `Names` and these information lives in the
host GHC).

GHC spawns a different `iserv` process depending on the selected target way:
`ghc-iserv-prof`, `ghc-iserv-dyn`, etc. This allows the `iserv` process to load
target object codes built which have not been built with the same way as GHC.

A different external interpreter can be specified with the `-pgmi` command-line
option.

Using the external interpreter in GHCi makes sense because it allows the
execution of the code produced for the target on the host (or remotely but it is
internal to the `iserv` process and GHC isn't aware of it).

Using the external interpreter to execute Template Haskell code doesn't really
make sense: TH code is similar to plugin code in that it has access to some
compiler internals (`Names`, etc.), it can modify the syntax tree and it can
perform IO (read files, etc.). Morally it should be built so that it can be
linked with the compiler and executed on the host.

Compiler plugins don't work at all with the external interpreter (see `#14335
<https://gitlab.haskell.org/ghc/ghc/issues/14335>`_). It is because they
directly depend on the `ghc` package and assume they are going to be linked with
it. Executing compiler plugins in the external interpreter would mean that the
communication protocol between iserv and GHC would need to be extended to
support everything a compiler plugin can do. As compiler plugins can do
virtually anything in the compiler, it would mean that most GHC datatypes would
need to be serializable, most functions explicitly exposed, etc. Moreover we
would have to deal with the discrepancy between host and target datatypes (word
size, etc.). It probably won't happen.

External interpreter links:

* https://gitlab.haskell.org/ghc/ghc/wikis/commentary/compiler/external-interpreter
* https://gitlab.haskell.org/ghc/ghc/wikis/remote-GHCi



Tasks
=====

Separate plugin packages/modules from target packages/modules
-------------------------------------------------------------

Currently GHC only considers one set of packages/modules: those for the target.
This is a problem because compiler plugins have to be compatible with GHC (same
way, same platform, etc.) but compiler plugins are looked for in target
packages/modules.

GHCJS `uses a hack
<https://github.com/ghcjs/ghcjs/blob/e87195eaa2bc7e320e18cf10386802bc90b7c874/src/Compiler/Plugins.hs#L2>`_ to
support plugins while its target is JavaScript code:
- the plugin still needs to exists amongst the target modules
- when loading a plugin module, instead of loading the plugin from the target
  database, it tries to find a matching module in the host database

The task is to make GHC aware of two databases: plugin and target. Loading a
plugin would be done via the plugin database and plugin would always be executed
with the internal interpreter.

Breaking change: currently GHC is able to compile its own plugins in confined
mode. In particular, it supports loading plugins from the "home package" (the
set of modules it is currently compiling). While GHC isn't multi-target, it
won't be able to build its own plugins. Cross-compilers such as GHCJS or
Asterius relies on two GHCs: one for the real target and one which targets the
compiler host (the latter is also used to build Cabal's Setup.hs files which are
run on the compiler host too).

Make GHC multi-target
---------------------

GHC should be able to produce code objects for at least 2 targets:

- its own host platform and compiler way (for plugins): `-target self`
- one or more other targets

We need a way to configure two toolchains (gcc, llvm, as, ld, ar, strip, etc.):
one for GHC plugins and another for the current target.


Make iserv program reinstallable
--------------------------------

Allow on-the-fly build of the iserv program. Depending on the selected target,
GHC should build an iserv program executing on the host (but not necessarily
with the same way as the compiler) that can execute target code.

GHC distributions wouldn't have to provide several `iserv` programs for every
target. They could be downloaded from Hackage and built for the host (now that
GHC would be multi-target).

Related issue: https://gitlab.haskell.org/ghc/ghc/issues/12218

Make boot libraries reinstallable
---------------------------------

GHC should be able to rebuild its boot libraries with different flags. Similarly
to iserv programs, GHC distributions shouldn't have to provide boot libraries
for every target (in addition to the boot libraries used by the compiler).

As plugin packages/modules would be separate from target packages/modules,
downloading boot libraries from Hackage and compiling them for the target
wouldn't impact plugin packages/modules.

Make GHC and the RTS reinstallable
----------------------------------

We also want GHC itself and the RTS to be reinstallable.

We should be able to specify the RTS package to use.

Related: https://gitlab.haskell.org/ghc/ghc/merge_requests/490

Blend ways into targets
-----------------------

Compiling for different compiler ways should be like cross-compiling for
different platforms. Compiler ways should be transformed into package flags for
the RTS and those flags should be stored into ABI hashes in installed packages
to avoid mismatching incompatible code objects.

These should be generic enough to allow different RTS options depending on the
selected RTS (e.g. native RTS should have flags equivalent to RTS ways,
Asterius/GHCJS RTS should have flags to select between NodeJS or browser targets
and to select features to enable).


Fix Template Haskell stage hygiene
----------------------------------

Currently Template Haskell mixes up stages because it assumes that the confined
mode is used.

We should be able to specify/detect if an `import` is for a top-level TH splice
or not.

We should remove `Lift` instances for target dependent types (e.g. `Word`,
`Int`, linux only types, etc.).

Related:

- see `this proposal <https://github.com/ghc-proposals/ghc-proposals/pull/243>`_
- `blog post
  <http://blog.ezyang.com/2016/07/what-template-haskell-gets-wrong-and-racket-gets-right/>`_


Don't use the external interpreter for Template Haskell
-------------------------------------------------------

Template Haskell code shouldn't be executed by the external interpreter but
similarly to plugins.

It should have dynamic access (i.e. not via CPP) to the target platform
properties (word size, endianness, etc.).

We should provide a way to query some stuff about the target code via the
external interpreter: e.g. `sizeOf (undefined :: MyStruct)`.

It should enhance speed as TH code is often used to perform syntactic
transformations (e.g.  `makeLenses`) which don't require target code evaluation.

Related: an alternative `proposal
<https://github.com/ghc-proposals/ghc-proposals/issues/162>`_ consists in
interpreting TH (target) code with a Core interpreter. However TH code may
invoke native functions which would be different depending on the target. We
really ought to execute/interpret GHC host code in all cases.

Cabal: Setup.hs
---------------

Cabal packages are built by a `Setup.hs` program running on the compiler host.
Most of them use the same "Simple" on but other use custom `Setup.hs`, with
dependencies specified in the `.cabal` files, etc.

Once GHC becomes multi-target, Stack and cabal-install could use `-target self`
to produce the actual program for the compiler host. It would ensure that the
compiler and `Setup` would use the same boot libraries.

Currently cross-compilers such as GHCJS and Asterius use two GHC compilers: one
for the target and another for the host (used to build the former, the plugins
and `Setup.hs` programs).

Cabal: `configure` build-type
-----------------------------

Some Cabal packages use `build-type: configure` (see the `user manual
<https://www.haskell.org/cabal/users-guide/developing-packages.html#system-dependent-parameters>`_).
During the configuration phase, the package description is modified by a
`configure` script producing a `buildinfo` file.

This only works on Unix-like systems and without additional parameters it
assumes that the target is the host.

Portable packages (in particular boot libraries) shouldn't use this. They might
call `configure` in custom `Setup.hs` on Unix-like platforms though, passing it
flags to specify the actual target if necessary.


Remove platform CPP
-------------------

GHC should expose a virtual package (like `ghc-prim`) with target information
(e.g. word size, endianness) as values/types instead of using CPP to include
`MachDeps.h`.

Expressions using these values would be simplified in Core.

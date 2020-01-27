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

GHC could build objects both for itself (i.e. using the way it has been built
with) and for the selected target way. By doing this, it could simulate the
confined mode by loading objects on the host that are different from the objects
produced for the target, with the hope that the two ways have no observable
difference. [I don't know if GHC does this. To check]

Another solution is to use the external interpreter.

Breach #3: external interpreter
-------------------------------

https://gitlab.haskell.org/ghc/ghc/wikis/commentary/compiler/external-interpreter
https://gitlab.haskell.org/ghc/ghc/wikis/remote-GHCi

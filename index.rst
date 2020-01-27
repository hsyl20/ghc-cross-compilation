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



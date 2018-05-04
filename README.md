<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License</a>.

Is this document missing something?
[Send a pull request or open an issue on GitHub!](https://github.com/fitzgen/requirements-for-wasm-debugging-capabilitiies)

--------------------------------------------------------------------------------

# Requirements for WebAssembly Debugging Capabilities

## Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [0. Preface](#0-preface)
- [1. General Requirements](#1-general-requirements)
  - [1.1. Must be future extensible](#11-must-be-future-extensible)
  - [1.2. Must be queryable without running the debuggee program](#12-must-be-queryable-without-running-the-debuggee-program)
  - [1.3. Must be embeddable within the WebAssembly](#13-must-be-embeddable-within-the-webassembly)
  - [1.4. Must be separable from the WebAssembly](#14-must-be-separable-from-the-webassembly)
  - [1.4. Must be compact on disk and over the network](#14-must-be-compact-on-disk-and-over-the-network)
  - [1.5. Should be compact in memory](#15-should-be-compact-in-memory)
  - [1.6. Should be fast to consume, generate, and manipulate](#16-should-be-fast-to-consume-generate-and-manipulate)
- [2. Required Capabilities](#2-required-capabilities)
  - [2.1. Locations](#21-locations)
    - [2.1.1. Must support querying the original source location for a given generated code location](#211-must-support-querying-the-original-source-location-for-a-given-generated-code-location)
    - [2.1.2. Must support querying the generated code location(s) for a given original source location](#212-must-support-querying-the-generated-code-locations-for-a-given-original-source-location)
    - [2.1.3. Must support enumerating all bidirectional mappings between original source and generated code locations](#213-must-support-enumerating-all-bidirectional-mappings-between-original-source-and-generated-code-locations)
  - [2.2 Inlined Functions](#22-inlined-functions)
    - [2.2.1. Should support querying the inline function frames that are logically on the stack at some generated code location](#221-should-support-querying-the-inline-function-frames-that-are-logically-on-the-stack-at-some-generated-code-location)
    - [2.2.2. Should support enumerating the logical inlined function invocations within a physical function](#222-should-support-enumerating-the-logical-inlined-function-invocations-within-a-physical-function)
  - [2.3 Types](#23-types)
    - [2.3.1. Should support describing scalar types](#231-should-support-describing-scalar-types)
    - [2.3.2. Should support describing compound types](#232-should-support-describing-compound-types)
  - [2.4 Scopes and Bindings](#24-scopes-and-bindings)
    - [2.4.1. Should support querying for the scope chain at a given generated code location](#241-should-support-querying-for-the-scope-chain-at-a-given-generated-code-location)
    - [2.4.2. Should support enumerating all bindings within a scope](#242-should-support-enumerating-all-bindings-within-a-scope)
    - [2.4.3. Should support querying a binding's type](#243-should-support-querying-a-bindings-type)
    - [2.4.4. Should support describing a method to find the location of or reconstruct a binding's value](#244-should-support-describing-a-method-to-find-the-location-of-or-reconstruct-a-bindings-value)
  - [2.5. Generic Functions and Monomorphizations](#25-generic-functions-and-monomorphizations)
    - [2.5.1. Should support querying whether a function is a monomorphization of a generic function](#251-should-support-querying-whether-a-function-is-a-monomorphization-of-a-generic-function)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## 0. Preface

This document attempts, as much as possible, to avoid embedding assumptions
about whether debugging capabilities are provided by parsing a static format (a
la DWARF), via a programmatic interface (a la language servers), a combination
of those two, or something else entirely. Rather than proposing a particular
solution, this document is an attempt to enumerate the requirements and
constraints that *any* solution must satisfy. Therefore, I use the term
"debugging capability" rather than "debugging information" or "debugger server".

Furthermore, I've categorized these requirements as "musts" or "shoulds". A
"must" is a hard requirement for an MVP: generally something that is already
supported by source maps, or is required _now_ in order to support a "should"
_later_. A "should" is something that is not a hard requirement for the MVP, but
for which we have supporting use cases.

Finally, here is an incomplete list of the kinds tools that would consume or
manipulate our hypothetical debugging capabilities:

* Stepping debuggers
* Logging consoles
* Time profilers (sampling- or tracing-based)
* Code size profilers (e.g. [Twiggy](https://github.com/rustwasm/twiggy))
* Linkers and bundlers (e.g. [`lld`](http://lld.llvm.org/) and
  [`webpack`](https://webpack.js.org/))
* Static analysis tools
* Instrumentation and rewriting tools (e.g. code coverage via instrumentation at
  the binary level)

This document's requirements were compiled with supporting these kinds tools in
mind.

## 1. General Requirements

### 1.1. Must be future extensible

So that we can ship incrementally, we need to be able to evolve the debugging
capabilities over time, as we add functionality. For example, we might initially
ship without support for describing packed struct types, but we must not
preclude ourselves from ever supporting them.

### 1.2. Must be queryable without running the debuggee program

For example, a CLI code size profiler must be able to consume information about
inlined functions or generic function monomorphization without running the
debuggee program. A static analysis tool that never runs its input program must
be able to report its results in terms of source locations. A linker or bundler
that is composing two wasm object files must not have to run the incomplete
programs contained in the object files in order to merge their debugging
capabilities.

### 1.3. Must be embeddable within the WebAssembly

For ease of development, the debugging capabilities must be embeddable within a
custom section or sections of the debuggee program's `.wasm` binary. This avoids
wrangling multiple files and associated issues like moving one file but not the
other and breaking relative references between them.

It is common to embed source maps as a data URL within `//# sourceMappingURL`
pragmas by base64 encoding them.

### 1.4. Must be separable from the WebAssembly

On the other hand, you most certainly do not want to include the debugging
capabilities within the `.wasm` binary you ship in production because of size
concerns and the associated network costs. But bugs happen in production, and
you might want to use a stepping debugger with your live production code base,
or apply debugging information to stack traces captured via telemetry and then
processed offline, a la [Sentry](https://sentry.io/welcome/).

### 1.4. Must be compact on disk and over the network

There is so much information that a tool might query for, on the order of the
debuggee program itself. Much of this information is regular and
repetitious. Source maps encode only a fraction of the information that we
ultimately want to support queries for, and they are regularly many megabytes in
size. The current source map specification is in its third revision, and each
revision has focused on making the format more dense. Empirically, relying on
general compression algorithms has not been enough.

### 1.5. Should be compact in memory

Ideally only portions of the debugging capabilities are necessary to load in
memory at any given time. E.g. if the debuggee program consisted of multiple
linked compilation units, then a debugger should only need to initially load the
capabilities for the current function's compilation unit. Answering a single
query (e.g. finding the original source location for a generated code location)
should not require loading the full debugging capabilities for the whole
debuggee program.

### 1.6. Should be fast to consume, generate, and manipulate

A fast tool provides a better user experience than a slow tool.

For example, a linker would ideally not need to eagerly apply relocs within the
debugging capabilities. Instead, the debugging capabilities consumers or
debugging capabilities themselves would lazily apply relocs when a particular
query forces it. The indexed variant of source maps provides similar lazy-reloc
functionality.

## 2. Required Capabilities

### 2.1. Locations

#### 2.1.1. Must support querying the original source location for a given generated code location

This is a common query. A trap is raised in the debuggee program and a stepping
debugger would like to know in which source location it should place its
cursor. A static analysis tool found something interesting at some location in
the generated code, and would like to present this to its user in terms of the
original source location. A profiler would like to express its sampled stack
trace in terms of original source locations.

#### 2.1.2. Must support querying the generated code location(s) for a given original source location

Another common query. A user sets a breakpoint on some original source line, and
the debugger must use the debugging capabilities to translate that into a set of
generated code locations that it should pause execution upon reaching. An
instrumentation tool that inserts tracepoints or logging at some original source
location would translate the input source location into a generated code
location where it should insert new instructions.

#### 2.1.3. Must support enumerating all bidirectional mappings between original source and generated code locations

Firefox's stepping debugger will enumerate all mappings to determine the set of
lines in the original source files that can have breakpoints set on them, and
highlight this information in the UI. Since pretty much every generated code
location ends up needing to be relocated when bundling JavaScript files,
bundlers will enumerate every mapping and apply relocs directly, rather than
maintaining a side table for these relocs.

### 2.2 Inlined Functions

#### 2.2.1. Should support querying the inline function frames that are logically on the stack at some generated code location

A sampling profiler should augment its sampled stack of physical frames with the
logical frames that were inlined. JavaScript profilers will do this today with
the help of the JavaScript engine, rather than using debugging capabilities.

#### 2.2.2. Should support enumerating the logical inlined function invocations within a physical function

What code ranges are due to an inlined function invocation? Note that a single
inlined function invocation may be spread across multiple, distinct ranges due
to code motion by the compiler.

Code size profilers want to find large functions that are getting aggressively
inlined many times, and might be causing size bloat.

### 2.3 Types

#### 2.3.1. Should support describing scalar types

Examples include signed and unsigned integers, floats, booleans, etc. What is
the name of the type? What is the type's size and alignment? These things are of
particular importance to stepping debuggers and anything that wishes to
understand a value in a running program's live memory.

#### 2.3.2. Should support describing compound types

Examples include pointers, arrays, structs, packed structs, structs with
bitfields, C-style enums, C-style untagged unions, Rust-style tagged unions,
etc. What is the name of the type? At what bit or byte offset is each member
field?

In addition to stepping debuggers, this information is also useful to tools that
are not actually running the debuggee program, like
[`ddbug`](https://github.com/gimli-rs/ddbug), which lets you inspect the sizes
and layouts of a program's types, so you can optimize their representation.

### 2.4 Scopes and Bindings

#### 2.4.1. Should support querying for the scope chain at a given generated code location

When a stepping debugger pauses at a location, it should display the scopes it
is paused within.

#### 2.4.2. Should support enumerating all bindings within a scope

When a stepping debugger is paused, it should display the bindings within each
scope it is paused within. Is the bindings a formal parameter? A local variable?
Is it a mutable or constant binding?

#### 2.4.3. Should support querying a binding's type

When a stepping debugger is paused, for each binding that is in scope, it should
be able to display the binding's type.

#### 2.4.4. Should support describing a method to find the location of or reconstruct a binding's value

When a stepping debugger is paused, for each binding it displays, it should also
display the binding's value.

"Reconstruct" because a value might not live in a single contiguous region of
memory or a single local. The compiler might have exploded it into its component
parts and passed each part by value in different parameters. It might be on the
stack during this bytecode range but in memory during that bytecode range within
the same function.

### 2.5. Generic Functions and Monomorphizations

#### 2.5.1. Should support querying whether a function is a monomorphization of a generic function

A code size profiler should identify generic functions that have been
monomorphized "too many" times and are leading to code bloat. They should be
able to do this regardless if the function is inlined or not.

# WebAssembly Debugging Capabilities

Currently, this document only lists motivation and requirements. It attempts, as
much as possible, to avoid embedding assumptions about whether debugging
capabilities are provided by parsing a static format (a la
[DWARF](http://dwarfstd.org/)), via a programmatic interface (a la [language
servers](https://github.com/Microsoft/language-server-protocol)), a combination
of those two, or by something else entirely. Rather than proposing a particular
solution, this is an attempt to enumerate the requirements and constraints that
*any* solution must satisfy.

[**📚 Read this document online! 📚**](http://fitzgen.github.io/wasm-debugging-capabilities)

## Building

Install [`bikeshed`](https://tabatkins.github.io/bikeshed/#installing) and then
run:

```
$ bikeshed spec index.bs index.html
```

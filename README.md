#  Musl Libc LLVM bitcode library


This is a fork of the [musl libc](http://www.musl-libc.org/).

It is a work in progress.

The idea is to produce a, complete as possible, LLVM bitcode version of
musl libc, motiviated in a simliar fashion to [Klee's uclibc](https://github.com/klee/klee-uclibc).

Our approach here will be to [port musl libc](http://wiki.musl-libc.org/wiki/Porting) to a new
architecture, based on the x86_64 version, but with all assembly replaced by llvm bitcode, where possible.

At the moment `--target=LLVM` behaves more like a `--no-asm`
switch, eliminating all the assembly for which there are vanilla C definitions.


## Recipe for libc.so.bc

Install [wllvm.](https://github.com/SRI-CSL/whole-program-llvm.git)
Then:

```
WLLVM_CONFIGURE_ONLY=1  CC=wllvm ./configure --target=LLVM --build=LLVM --prefix=<install dir>
make
cd lib
extract-bc --bitcode libc.a
cp libc.a.bc  <wherever you want your static bitcode library to live>
extract-bc libc.so
cp libc.so.bc  <wherever you want your shared bitcode library to live>
```
Note the `--bitcode` option is a recent addition to `wllvm` and currently
only available in [SRI's fork](https://github.com/SRI-CSL/whole-program-llvm),
I will eventually push it up stream, but have not yet. 

## Ian's notes on what to do with the bitcode

Suppose you have an application built,  `nweb.bc` say.  
Then you can do:
```
clang -static -nostdlib nweb.bc libc.a.bc crt1.o libc.a -o nweb
```
or
```
llc -filetype=obj nweb.bc
llc -filetype=obj libc.a.bc
clang -static -nostdlib nweb.o libc.a.o crt1.o libc.a -o nweb
```
and get a working executable. The use of `llc` is optional in principle,
but my `clang` (3.5) crashes on some large `.bc` files. You will find
`crt1.o` and  `libc.a` in the same directory you produced `libc.a.bc`.

The `crt1.o` is needed to provide the definition of the entry symbol `__start`.
While the `libc.a` archive is required to provide the definitions of those things
that do not have `C` (or `C++`) definitions, i.e. are written in
assembly. Examples of the later are `__clone`, `__syscall`, `setjmp` and `longjmp`.

**Note Bene:** the above works for `llvm-3.5`, for `llvm-3.8` you need to `locate`
your system `libgcc.a` and add that to the list. When I figure out how to use 
`compiler-rt` to get around that I will mention it here.  OK so I finally figured 
out the `compiler-rt` thing.  I built `llvm-3.8.1` using `wllvm` and `wllvm++`
together with the projects `clang`, `compiler-rt`, `libcxx`, `libcxxabi`, and `lld`, which 
turned out to be pretty painless. One of the build products was `libclang_rt.builtins-x86_64.a`,
from which I was able to extract `libclang_rt.builtins-x86_64.bc` using the command:
```
extract-bc -b libclang_rt.builtins-x86_64.a
```
This bitcode module has definitons for those pesky instrinsics like
`__mulxc3`, `__mulsc3`, and `__muldc3`.

Of course all this effort is not very interesting unless you have some fun
with the bitcode before this final linking phase.

You can also do things like:

```
llvm-link nweb.bc libc.a.bc -o nweb_app.bc
opt -O3 nweb_app.bc -o nweb_app_o3.bc
llc -filetype=obj nweb_app_o3.bc
clang -static -nostdlib nweb_app.o crt1.o libc.a -o nweb
```

## Ian's notes on what NOT to do with the bitcode


I guess pointing out things that don't work might also illucidate things, especially
if I can explain why they don't work.
This does not work:
```
clang -static -nostdlib nweb.bc libc.so.bc crt1.o libc.a -o nweb
```
It links without warnings, but crashes on running. The reason it crashes on running
is that `libc.so.bc` contains an empty definition of the thread local storage
initializer, `__init_tls`,
which it gets from `ldso/dynlink.c`. The actual definition that is needed is
`static_init_tls` which is a weak alias defined in `src/env/__init_tls.c`.
The static library `libc.a` is set up correctly:
```
> nm libc.a | grep init_tls
__init_tls.o:
0000000000000000 W __init_tls
0000000000000000 t static_init_tls
                 U __init_tls
```
whereas the Frankenstein
one gets by linking a dynamic object `libc.so.bc` with a static object `libc.a` is not. The `SIGSEGV`
occurs because of this problematic uninitialized thread local storage.


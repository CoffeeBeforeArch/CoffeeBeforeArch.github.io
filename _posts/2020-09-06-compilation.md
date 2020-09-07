---
layout: default
title: Compilation with GCC
---

# Compilation with GCC

Many programmers coloquially refer to the process of generating an executable from high-level source code as compilation, and refer to `g++` as a compiler. In reality, `g++` is a compiler driver that invokes different subcomponents of GCC (the GNU Compiler Collection) that work together to generate an executable. This includes a preprocessor, compiler, assembler, and linker. In this blog post, we'll be going over each of these subcomponents for GCC, and analyzing how our source code is transformed by each step.

### Relevant Links

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## A "Hello, world!" Program

The program we'll be anaylzing in this post is one that prints `Hello, world!` to the screen. Here is the exact code we'll be using:

```cpp
// A simple "Hello, world!" program written in C++
// By: Nick from CoffeeBeforeArch

#include <iostream>

int main() {
  // Print to the screen
  std::cout << "Hello, world!\n";
  
  // Return that the program completed successfully
  return 0;
}
```

To generate an executable, we can use the compiler driver `g++` in the with the following commands:

```bash
g++ hello_world.cpp -o hello_world
```

We can execute the program using `./hello_world`, which will cause `"Hello, world!"` to be printed to the screen.

```text
cba@cba:~/forked_repos/CoffeeBeforeArch.github.io/_posts$ ./hello_world 
Hello, world!
```

Now that we've looked at how to generate an executable, let's look at all the intermediate steps.

## Preprocessing

The first step for generating an executable with GCC is preprocessing. In the preprocessing stage, `g++` invokes `cpp`, The C Preprocessor. This does things like:

- Expand macros
- Include header files
- Enable/Disable parts of code

For our simple `Hello, world!` program, the the preprocessor will take care of the line `#include <iostream>`, and replace it the code from the header files for the stream-based I/O library.

We can do preprocessing in two ways. The first is by directly invoking `cpp`:

```bash
cpp hello_world.cpp -o hello_world.ii
```

We can also use the compiler driver `g++` with the `-E` flag to specify we want to stop after preprocessing:

```bash
g++ -E hello_world.cpp -o hello_world.ii
```

The resulting output file (`hello_world.ii`) contains over 30,000 lines of code! Let's take a look at some of the relevant parts. First, we have our original `main` function:

```cpp
# 6 "hello_world.cpp"
int main() {

  std::cout << "Hello, world!\n";


  return 0;
}
```

The only major difference is that our comments have been stripped out.

The remaining ~30,000 lines are what replaced the `#include <iostream>` directive in our original program. Within these lines, we can find reference to `std::cout`:

```cpp
namespace std __attribute__ ((__visibility__ ("default")))
{

# 60 "/usr/include/c++/8/iostream" 3
  extern istream cin;
  extern ostream cout;
  extern ostream cerr;
  extern ostream clog;


  extern wistream wcin;
  extern wostream wcout;
  extern wostream wcerr;
  extern wostream wclog;




  static ios_base::Init __ioinit;


}

```

Within `namespace std`, we see `extern ostream cout;`. This line specifies that the definition for our `cout` `ostream` object exists someplace else, and will be taken care of in later stages of compilation.

## Compilation

Now that the preprocessor has fetched the code we needed from the `iostream` header files, we can move on to compilation. Compilation is the process of translating our high-level source into assembly. The GCC compiler for C++ is `cc1plus`.

While we can invoke `cc1plus` ourselves, it is not advised. Instead, we will pass our preprocessed output (`hello_world.ii`) to `g++`, and use the `-S` flag to say we want to stop after compilation.

```bash
g++ -S hello_world.ii -o hello_world.s
```

Let's take a look at a few different parts of the assembly to see what's going on. First, we can find our string `"Hello, world!\n"` in a section called `.rodata`:

```assembly
	.section	.rodata
	.type	_ZStL19piecewise_construct, @object
	.size	_ZStL19piecewise_construct, 1
_ZStL19piecewise_construct:
	.zero	1
	.local	_ZStL8__ioinit
	.comm	_ZStL8__ioinit,1,1
.LC0:
	.string	"Hello, world!\n"
```

The `.rodata` section is for read-only data, and our string can be found under the label `.LC0`.

We can also find our `main` function:

```assembly
main:
	endbr64
	pushq	%rbp
	movq	%rsp, %rbp
	leaq	.LC0(%rip), %rsi
	leaq	_ZSt4cout(%rip), %rdi
	call	_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@PLT
	movl	$0, %eax
	popq	%rbp
	ret
```

Ignoring the prologue and epilogue that manages the stack, the first thing our program dies is load the address for our string from the label `.LC0` using a load effect address (`lea`) instruction:

```assembly
	leaq	.LC0(%rip), %rsi
```

We also see that we load the address of `cout`, and call a long mangled name that includes `basic_ostream` inside it:

```assembly
	leaq	_ZSt4cout(%rip), %rdi
	call	_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@PLT
```

This is really just a call to the `operator::<<`.

The final thing our program does is our `return 0;` statement. It does this by setting the `%eax` register to `0` using `movl`, then return using `ret`:

```assembly
	movl	$0, %eax
	popq	%rbp
	ret
```

Note `popq` is just part of managing the stack.

## Assembly

Our instructions must be translated into machine code by an assembler before our processor can read them. The assembler for GCC is `as` (The GNU Assembler).

We can run the GNU Assembler directly by invoking `as`:

```bash
as hello_world.s -o hello_world.o
```

We can also use `g++` with the `-c` flag to say we want to stop after assembly:

```bash
g++ -c hello_world.s -o hello_world.o
```

The result is typically referred to object code or object files (with the `.o` extension).

We can translate our object code back to human-readable assembly using the utility `objdump.` For example, we can run the following command that tells `objdump` to disassemble our object file with `-d`, and remove the C++ name mangling using `-C`:

```bash
objdump -d -C hello_world.o
```

By default, this will only dump `.text` sections, which contain the instructions of our program.

Let's start by looking at our `main` function:

```assembly
hello_world.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:	f3 0f 1e fa          	endbr64 
   4:	55                   	push   %rbp
   5:	48 89 e5             	mov    %rsp,%rbp
   8:	48 8d 35 00 00 00 00 	lea    0x0(%rip),%rsi        # f <main+0xf>
   f:	48 8d 3d 00 00 00 00 	lea    0x0(%rip),%rdi        # 16 <main+0x16>
  16:	e8 00 00 00 00       	callq  1b <main+0x1b>
  1b:	b8 00 00 00 00       	mov    $0x0,%eax
  20:	5d                   	pop    %rbp
  21:	c3                   	retq   
```

We now have our instructions and their hex encoding side-by-side. One important thing to note is that many of our addresses, including those for our `"Hello, world!\n"` string and `cout` have been replaced by placeholders. To understand why, we need to briefly talk about the symbol table.

The symbol table is a map between names (like `cout`) and information related to those names. If we dump the symbol table for our object code using `nm -C hello_world.o`, we get the following output (note, `-C` gets rid of the C++ name mangling):

```text
                 U __cxa_atexit
                 U __dso_handle
                 U _GLOBAL_OFFSET_TABLE_
000000000000006f t _GLOBAL__sub_I_main
0000000000000000 T main
0000000000000022 t __static_initialization_and_destruction_0(int, int)
                 U std::ios_base::Init::Init()
                 U std::ios_base::Init::~Init()
                 U std::cout
0000000000000000 r std::piecewise_construct
0000000000000000 b std::__ioinit
                 U std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)
```

For each symbol, we get an address, and the symbol type. Many of our symbols are of the `U` type. These means the symbols are undefined, and will need to be resolved in the next phase (linking). Other symbols are of the `T` or `t` type, meaning they live in the text (code) section of the object file. Type `r` symbols are from the read-only data section (like our string), and `b` symbols are in the BSS data section (zero/uninitialized data).

## Linking

The final step of generating an executable is linking. This is where our object files get linked together, and our placeholder addresses get replaced with their final values. The linker for GCC is `ld`. However, GCC will use the `collect2` utility which is a wrapper around `ld`.

We will finish linking our program with `g++`. By default, `g++` will link against things like the C++ standard library implementation, `libstdc++.so`. Here is the final command we'll use:

```bash
g++ hello_world.o -o hello_world
```

This gives us a fully formed executable! Let's dump the symbol table to see how things have changed after linking:

```text
0000000000004010 B __bss_start
0000000000004150 b completed.0
                 U __cxa_atexit@@GLIBC_2.2.5
                 w __cxa_finalize@@GLIBC_2.2.5
0000000000004000 D __data_start
0000000000004000 W data_start
00000000000010d0 t deregister_tm_clones
0000000000001140 t __do_global_dtors_aux
0000000000003d98 d __do_global_dtors_aux_fini_array_entry
0000000000004008 D __dso_handle
0000000000003da0 d _DYNAMIC
0000000000004010 D _edata
0000000000004158 B _end
0000000000001298 T _fini
0000000000001180 t frame_dummy
0000000000003d88 d __frame_dummy_init_array_entry
00000000000021ac r __FRAME_END__
0000000000003fa0 d _GLOBAL_OFFSET_TABLE_
00000000000011f8 t _GLOBAL__sub_I_main
                 w __gmon_start__
0000000000002014 r __GNU_EH_FRAME_HDR
0000000000001000 t _init
0000000000003d98 d __init_array_end
0000000000003d88 d __init_array_start
0000000000002000 R _IO_stdin_used
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
0000000000001290 T __libc_csu_fini
0000000000001220 T __libc_csu_init
                 U __libc_start_main@@GLIBC_2.2.5
0000000000001189 T main
0000000000001100 t register_tm_clones
00000000000010a0 T _start
0000000000004010 D __TMC_END__
00000000000011ab t __static_initialization_and_destruction_0(int, int)
                 U std::ios_base::Init::Init()@@GLIBCXX_3.4
                 U std::ios_base::Init::~Init()@@GLIBCXX_3.4
0000000000004040 B std::cout@@GLIBCXX_3.4
0000000000002004 r std::piecewise_construct
0000000000004151 b std::__ioinit
                 U std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)@@GLIBCXX_3.4
```

We now see that our symbol table has grown quite a bit, and symbols like `cout` that were undefined now have final addresses. Let's use `objdump` to see how our assembly has changed:

```assembly
0000000000001189 <main>:
    1189:	f3 0f 1e fa          	endbr64 
    118d:	55                   	push   %rbp
    118e:	48 89 e5             	mov    %rsp,%rbp
    1191:	48 8d 35 6d 0e 00 00 	lea    0xe6d(%rip),%rsi        # 2005 <std::piecewise_construct+0x1>
    1198:	48 8d 3d a1 2e 00 00 	lea    0x2ea1(%rip),%rdi        # 4040 <std::cout@@GLIBCXX_3.4>
    119f:	e8 dc fe ff ff       	callq  1080 <std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)@plt>
    11a4:	b8 00 00 00 00       	mov    $0x0,%eax
    11a9:	5d                   	pop    %rbp
    11aa:	c3                   	retq   

```

Inside our `main` function, we see that our placeholder addresses for our string and `cout` have been replaced by real values.

The address `4040` in the comment at the end of the `lea` instruction for `std::cout` corresponds to the same address we dumped from the symbol table:

```text
0000000000004040 B std::cout@@GLIBCXX_3.4
```

We also can dump the `.rodata` section from the executable as we did with our object code:

```text
hello_world:     file format elf64-x86-64

Contents of section .rodata:
 2000 01000200 0048656c 6c6f2c20 776f726c  .....Hello, worl
 2010 64210a00                             d!..            
```

Note, the address `2005` in the comment at the end of the `lea` instruction for our string corresponds to the start of our string in the `.rodata` section (`2000 + 5`). This address is calculated in the comment with `<std::piecewise_construct+0x1>`, where `std::piecewise_construct` is located at `2004`, as seen in the symbol table:

```
text
0000000000002004 r std::piecewise_construct
```

## Additional Notes

### Generating Intermediate Files

If you want `g++` to save the result from all the intermediate steps of compilation, you can pass it the `-save-temps` flag:

```bash
g++ hello_world.cpp -o hello_world -save-temps
```

This will generate the preprocessed output `hello_world.ii`, compiled code `hello_world.s`, object code `hello_world.o`, and executable `hello_world`.

### Verbose Output from g++

You can also get the commands that the compiler drive `g++` used to compile your application by specifying the `-v` option. Here's the output from my machine running `g++ hello_world.cpp -o hello_world -v`:

```bash
Using built-in specs.
COLLECT_GCC=g++
COLLECT_LTO_WRAPPER=/usr/local/libexec/gcc/x86_64-pc-linux-gnu/10.0.1/lto-wrapper
Target: x86_64-pc-linux-gnu
Configured with: ../configure --disable-multilib
Thread model: posix
Supported LTO compression algorithms: zlib
gcc version 10.0.1 20200412 (experimental) (GCC) 
COLLECT_GCC_OPTIONS='-o' 'hello_world' '-v' '-shared-libgcc' '-mtune=generic' '-march=x86-64'
 /usr/local/libexec/gcc/x86_64-pc-linux-gnu/10.0.1/cc1plus -quiet -v -imultiarch x86_64-linux-gnu -D_GNU_SOURCE hello_world.cpp -quiet -dumpbase hello_world.cpp -mtune=generic -march=x86-64 -auxbase hello_world -version -o /tmp/cc9kgEvV.s
GNU C++14 (GCC) version 10.0.1 20200412 (experimental) (x86_64-pc-linux-gnu)
	compiled by GNU C version 10.0.1 20200412 (experimental), GMP version 6.1.0, MPFR version 3.1.4, MPC version 1.0.3, isl version isl-0.18-GMP

GGC heuristics: --param ggc-min-expand=30 --param ggc-min-heapsize=4096
ignoring nonexistent directory "/usr/local/include/x86_64-linux-gnu"
ignoring nonexistent directory "/usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/../../../../x86_64-pc-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/../../../../include/c++/10.0.1
 /usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/../../../../include/c++/10.0.1/x86_64-pc-linux-gnu
 /usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/../../../../include/c++/10.0.1/backward
 /usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/include
 /usr/local/include
 /usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/include-fixed
 /usr/include/x86_64-linux-gnu
 /usr/include
End of search list.
GNU C++14 (GCC) version 10.0.1 20200412 (experimental) (x86_64-pc-linux-gnu)
	compiled by GNU C version 10.0.1 20200412 (experimental), GMP version 6.1.0, MPFR version 3.1.4, MPC version 1.0.3, isl version isl-0.18-GMP

GGC heuristics: --param ggc-min-expand=30 --param ggc-min-heapsize=4096
Compiler executable checksum: 46de3d8a6d088bac0288614969e77d24
COLLECT_GCC_OPTIONS='-o' 'hello_world' '-v' '-shared-libgcc' '-mtune=generic' '-march=x86-64'
 as -v --64 -o /tmp/ccnpWHbO.o /tmp/cc9kgEvV.s
GNU assembler version 2.30 (x86_64-linux-gnu) using BFD version (GNU Binutils for Ubuntu) 2.30
COMPILER_PATH=/usr/local/libexec/gcc/x86_64-pc-linux-gnu/10.0.1/:/usr/local/libexec/gcc/x86_64-pc-linux-gnu/10.0.1/:/usr/local/libexec/gcc/x86_64-pc-linux-gnu/:/usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/:/usr/local/lib/gcc/x86_64-pc-linux-gnu/
LIBRARY_PATH=/usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/:/usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/../../../../lib64/:/lib/x86_64-linux-gnu/:/lib/../lib64/:/usr/lib/x86_64-linux-gnu/:/usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-o' 'hello_world' '-v' '-shared-libgcc' '-mtune=generic' '-march=x86-64'
 /usr/local/libexec/gcc/x86_64-pc-linux-gnu/10.0.1/collect2 -plugin /usr/local/libexec/gcc/x86_64-pc-linux-gnu/10.0.1/liblto_plugin.so -plugin-opt=/usr/local/libexec/gcc/x86_64-pc-linux-gnu/10.0.1/lto-wrapper -plugin-opt=-fresolution=/tmp/ccLOxCSG.res -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lgcc --eh-frame-hdr -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o hello_world /usr/lib/x86_64-linux-gnu/crt1.o /usr/lib/x86_64-linux-gnu/crti.o /usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/crtbegin.o -L/usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1 -L/usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/../../../../lib64 -L/lib/x86_64-linux-gnu -L/lib/../lib64 -L/usr/lib/x86_64-linux-gnu -L/usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/../../.. /tmp/ccnpWHbO.o -lstdc++ -lm -lgcc_s -lgcc -lc -lgcc_s -lgcc /usr/local/lib/gcc/x86_64-pc-linux-gnu/10.0.1/crtend.o /usr/lib/x86_64-linux-gnu/crtn.o
COLLECT_GCC_OPTIONS='-o' 'hello_world' '-v' '-shared-libgcc' '-mtune=generic' '-march=x86-64'
```

### Shared Libraries

One details we did not cover in detail in this example is shared libraries. Check out [this](https://amir.rachum.com/blog/2016/09/17/shared-libraries/) blog post on shared libraries and dynamic loading for more details.

## Final Thoughts

Understanding the individual components of compilation can be useful when debugging complex compilation errors and trying to speed up the build time of large applications. It's also a great place to start if you want to start working on compilers yourself.

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


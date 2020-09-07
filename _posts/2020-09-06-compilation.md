---
layout: default
title: Compilation with GCC
---

# Compilation with GCC

Many programmers coloquially refer to the process of generating an executable from high-level source code as compilation, and refer to `g++` as a compiler. In reality, `g++` is a compiler driver that invokes numerous subcomponents of GCC (the GNU Compiler Collection) that work together to generate an executable. This includes a preprocessor, compiler, assembler, and linker. In this blog post, we'll be going over each of these subcomponents for GCC, and analyzing how our source code is transformed by each step.

### Relevant Links

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com

## A "Hello, world!" Program

The program we'll be anaylzing in this post is the classical `Hello, world!` example. That will look something like this:

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

We can execute the program using `./hello_world`, and you should see `"Hello, world!"` printed to the screen.

## Preprocessing

The first step for generating an executable with GCC is preprocessing. In the preprocessing stage, `g++` invokes the preprocessor which does things like:

- Expanding macros
- Including header files
- Enabling/Disabling parts of code

The pre-processor for GCC is `cpp` (The C Preprocessor). For our simple `Hello, world!` program, the primary job of the preprocessor will be fetching the the header files for the stream-based I/O library `iostream`.

We can invoke the preprocessor in two different ways. We can invoke the preprocessor direcly using the following:

```bash
cpp hello_world.cpp -o hello_world.ii
```

We can also use the compiler driver `g++` with the `-E` flag to specify we want to only perform preprocessing:

```bash
g++ -E hello_world.cpp -o hello_world.ii
```

The resulting output file is of 30,000 lines long! Let's take a look at some of the relevant parts. First, we have our original `main` function:

```cpp
# 6 "hello_world.cpp"
int main() {

  std::cout << "Hello, world!\n";


  return 0;
}
```

The only major difference is that our comments have been stripped out.

Within the remaining 30,000 lines from included from `#include <iostream>`, we can find the following:

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

Within `namespace std`, we see `extern ostream cout;`. This line specifies that the definition for our `cout` `ostream` object exists someplace else.

## Compilation

Now that the preprocessor has fetched and pasted the code for the `iostream` header files into our source file, we can move on to compilation. Compilation is the process of translating our high-level source into assembly. The GCC compiler for C++ is `cc1plus`.

While we can invoke `cc1plus` ourselves. it is not advised. Instead, we will pass our preprocessed output (`hello_world.ii`) to `g++`, and use the `-S` flag to say we want to stop after compilation.

```bash
g++ -S hello_world.ii -o hello_world.s
```

Let's take a look at a few different of the assembly to see what's going on. First, we can find our string `"Hello, world!\n"` in a section called `.rodata`:

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

We can also find our main function:

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

First we can see that our program loads the address from our string from the label `.LC0` using a load effect address (`lea`) instruction:

```assembly
	leaq	.LC0(%rip), %rsi
```

We also see that we load the address of `cout`, and call a long mangled name that includes `basic_ostream` inside it:

```assembly
	leaq	_ZSt4cout(%rip), %rdi
	call	_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@PLT
```

This is how our string gets printed.

The final thing our program does is our `return 0;` statement. It does this by setting the `%eax` register to `0`, restoring the frame pointer (`%rbp`) and then return using `ret`:

```assembly
	movl	$0, %eax
	popq	%rbp
	ret
```

## Assembly

Before our processor can understand these instructions, they must be translated into machine code by an assembler. The assembler for GCC is GAS (The GNU Assembler). We can run the gnu assembler directly by invoking `as`, or through `g++` using the `-c` flag.

Here is how that is done with `as`:

```bash
as hello_world.s -o hello_world.o
```

And here is how we can do the same with with `g++`:

```bash
g++ -c hello_world.s -o hello_world.o
```

The resulting `.o` file are known as object code or object files.

We can translate our object code back to human-readable assembly using a utility like `objdump.` For example, we can run the following command that tells `objdump` to disassemble our object file with `-d`, and remove the C++ name mangling using `-C`. By default, this will only dump `.text` sections, which contain the instructions of our program:

```bash
objdump -d -C hello_world.o
```

Here is our main function:

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

We can now see our instructions with their hex encoding side-by-side. One other thing to note is that many of our addresses, including those for our `"Hello, world!\n"` string and `cout` have been replaced by placeholders. This leads us to the symbol table.

The symbol table is a map between a name (like `cout`) and infromation related to that name. If we dump the symbol table for our object code (using `nm -C hello_world.o`), we get the following:

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

For each symbol, we get an address, and the symbol type. Many of our symbols are of the `U` type. These means the symbols are undefined, and will need to be resolved in the next phase (linking). Other sybols are of the `T` or `t` type, meaning they live in the text (code) section of the object file. Type `r` symbols were found in the read-only data section, and `b` symbols are in the BSS data section (zero/uninitialized data).

## Linking

The final step of generating an executable is linking. This is where our object files get linked together, and all our addresses get finalized. The linker for GCC is `ld`. However, GCC will use the `collect2` utility, which is a wrapper over `ld`.

We can finish linking our program with GCC.

```bash
g++ hello_world.o -o hello_world
```

This will give use a fully formed executable! Let's dump the symbol table to see how things have changed after linking:

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

We now see that our sybol table has grown, and our symbols like `cout` now have a final address. Let's use `objdump` to see how our assembly has changed:

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

Inside of our main function, we see that our placeholder addresses for our `string` and `cout` have been replaced by real values. We also can dump the `.rodata` section from the executable:

```text
hello_world:     file format elf64-x86-64

Contents of section .rodata:
 2000 01000200 0048656c 6c6f2c20 776f726c  .....Hello, worl
 2010 64210a00                             d!..            
```

## Final Thoughts

Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


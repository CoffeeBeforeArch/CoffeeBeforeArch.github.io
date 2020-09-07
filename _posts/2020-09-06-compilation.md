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

We can now see our instructions with their hex encoding side-by-side. One other thing to note is that many of our addresses, including those for our `"Hello, world!\n"` string and `cout` have been replaced by placeholders.

## Linking

## Final Thoughts


Thanks for reading,

--Nick

### Link to the source code

- [My YouTube Channel: ](https://www.youtube.com/channel/UCsi5-meDM5Q5NE93n_Ya7GA?view_as=subscriber)
- [My GitHub Account: ](https://github.com/CoffeeBeforeArch)
- My Email: CoffeeBeforeArch@gmail.com


---
layout: post
categories: [ "Programming language" ]

title: Handle C++ exceptions in C code, part 1
---

Hold on! Before you call me crazy, please let me clarify. Yes, I am going to tell you how to handle C++ exceptions in C code,
but don't worry, I know this is a terrible idea, and (hopefully) no one is _actually_ going to do that.

But why, you might ask, would anyone even begin to ponder such a silly idea? Well, poking things and making them do things they
aren't _supposed to_ do is a brilliant way to figure out how they work. And exception handling is certainly a very interesting
, and perhaps quite important, practical computer algorithm. So it's quite natural for someone to have curiosity about how it works.

Alright. But where do we even start? If you search for information on exception handling online, what you would find is
usually cryptic and incomplete documentation about ELF, Dwarf, and C++ ABI. Often they are difficult to understand, and it
is usually hard for someone to see how they all fit together.

So instead, let's try to work in reverse instead. Let's try to inspect something that actually does exception handling.

```cpp
int throw_exception() {
    throw nullptr;
}
int main() {
    try {
        throw_exception();
    } catch (void *e){
    }
}
```

This is a baseline program that throws an exception and catches it. Let's try compiling it and see what comes out:

```bash
g++ code.cc -S
```

If you are wondering what `-S` does, it tells the compiler to stop after producing the assembly, without assembling or linking the code.
You should try this step yourself, and look at the assembly produced. Here, I picked out a few relevant snippets, and simplified them to
make them easier to read.

Let's look at the throwing side first:

```nasm
throw_exception():
.LFB0:
	push	rbp
	mov	rbp, rsp
	mov	edi, 8
	call	__cxa_allocate_exception@PLT ; <----
	mov	QWORD PTR [rax], 0
	mov	edx, 0
	lea	rsi, _ZTIDn[rip]
	mov	rdi, rax
	call	__cxa_throw@PLT              ; <----
```

If you are not used to looking at assembly, this might look quite daunting. But luckily, you don't have to know any assembly at all to pick out the interesting bits here:
`__cxa_allocate_exception` and `__cxa_throw`.

As their names suggest, `__cxa_allocate_exception` allocates space for the thrown exception, and `__cxa_throw` actually commences the throwing.
(the `__cxa_` bit probably means "cxx abi", I am not sure.) You don't have to take my words, you can find this information [here](https://libcxxabi.llvm.org/spec.html).

If you are curious, you can find the code of `__cxa_throw` [here](https://github.com/gcc-mirror/gcc/blob/a16da48bf19bb139e5461e5b5b7f072d5369b054/libstdc%2B%2B-v3/libsupc%2B%2B/eh_throw.cc#L75).
For the rest of us, we are move on to the catching side:

```nasm
main:
.LFB1:
	push	rbp
	mov	rbp, rsp
	sub	rsp, 16
.LEHB0:
	call	throw_exception()
.LEHE0:
.L6:
	mov	eax, 0
	jmp	.L8 ; <- this will jump to the label below
.L7:
	cmp	rdx, 1
	je	.L5
	mov	rdi, rax
.LEHB1:
	call	_Unwind_Resume@PLT
.LEHE1:
.L5:
	mov	rdi, rax
	call	__cxa_begin_catch@PLT
	mov	QWORD PTR -8[rbp], rax
	call	__cxa_end_catch@PLT
	jmp	.L6
.L8: ; <- this is a label
	leave
	ret ; <- this means "return"
```

(If you don't know how to read this assembly, you just need to know this: instructions which start with letter `j` mean jump. The thing which follows the jump instruction is a "label", which is also what the `.Lxxx:` you see in the code are.
The jump instruction will jump to the matching label when executed)

OK, here we spot `__cxa_begin_catch` and `__cxa_end_catch`, which seems to be relevant. However, they don't seem to be directly reachable. If you follow the call to `throw_exception()`, you can see after it returns, the code immediately jumps
to `.L8`, then returns from `main`. So who is going to execute the `catch` block?

Let's look a bit further into the assembly:

```nasm
.LFE1:
        .globl  __gxx_personality_v0
        .section        .gcc_except_table,"a",@progbits
        .align 4
.LLSDA1:
        .byte   0xff
        .byte   0x9b
        .uleb128 .LLSDATT1-.LLSDATTD1
.LLSDATTD1:
        .byte   0x1
        .uleb128 .LLSDACSE1-.LLSDACSB1
.LLSDACSB1:
        .uleb128 .LEHB0-.LFB1
        .uleb128 .LEHE0-.LEHB0
        .uleb128 .L7-.LFB1 ; <- ha
        .uleb128 0x1
        .uleb128 .LEHB1-.LFB1
        .uleb128 .LEHE1-.LEHB1
        .uleb128 0
        .uleb128 0
```

This assembly block defines _data_. After being assembled and linked, they become data bytes stored verbatim in the executable. And in there, we find a reference to `.L7`.
Look back at the code, we can see `.L7` contains a jump to `.L5`, which is the `catch` block.

From this, we can infer that, there is some information stored in the executable ("metadata"), which the exception throwers will read when they throw an exception, to figure out what code to run when they throw an exception.

Alright, we have a <span id="rough" class="anchor">rough</span> picture of how exception handling works:

* The thrower prepares the exception, then call `__cxa_throw`
* `__cxa_throw` looks through the metadata of the callers in the function call stack, to find a suitable catcher
* The program jumps to the catcher

Looks simple, right? If we can somehow make the metadata of a _C_ function yell: "hey, look here, I am an exception catcher!", we will be done here.

But, as you might know, there is no such thing as exceptions in the C language. There can't possibly be a way to declare a function as exception catcher, right?

Well, there is not a _standard_ way. But there is a way. There is this extension to C, implemented by the GCC compiler (and later Clang), which allows you to attach a
["cleanup"](https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html#:~:text=cleanup) function to a local variable. This cleanup function will be called when
the local variable goes out of scope.

Because of the excellent interoperability between C and C++, you can have the situation where an C++ exception is thrown "through" a C function. Normally, the cleanup functions
aren't going to run in that case. However, this makes people unhappy. So, now, there _is_ a way to make them run.

Passing the `-fexceptions` option to GCC, makes it generate exception-aware code even when compiling C code. And in that mode, the cleanup functions will be registered as exception handlers! Just like the destructors of local variables in C++.

Here is an example:

```cpp
// file1.cc
extern "C" void throw_exception() {
    throw nullptr;
}
extern  "C" void might_throw();
int main() {
    try {
        // OK, we have to have a try/catch in main,
        // because if C++ cannot find an actual `catch` block,
        // it will just abort(), without going through the throw
        // process at all.
        might_throw();
    } catch (void *e) {
    }
}
```

```c
// file2.c
#include <stdio.h>
extern void throw_exception();
void a_cleanup_function(void *x) {
    fprintf(stderr, "cleanup called\n");
}
void might_throw() {
    // Call a_cleanup_function when `x` goes out of scope
    __attribute__((cleanup(a_cleanup_function))) int x;
    throw_exception();
    fprintf(stderr, "do more stuff\n"); // we don't get to this point :'(
}
```

Compile and run:

```bash
g++ file1.cc -c
gcc file2.c -fexceptions -c
g++ file1.o file2.o
./a.out
```

You will see it says:

```plain
cleanup called
```

If you look at the assembly of `file2.c`, you will see structures very similar to the `try/catch` block in `file1.cc`.

OK, we are able to get a C function called during the handling of a _C++_ exception. But this exception still propagated through the C function, why?

Let's look at the assembly again:

```nasm
might_throw:
; ... skipped ...
.L7:
	mov	rbp, rax
	jmp	.L5
.L5:
	lea	rdi, 4[rsp]
.LEHB2:
	call	a_cleanup_function
	mov	rdi, rbp
	call	_Unwind_Resume@PLT
; ... more skipped ...
```

Here you can easily see the call to `a_cleanup_function`, right after that, is a call to `_Unwind_Resume`. What does that do?

First of all, what does "unwind" mean here? You might have already known, the program needs to "unwind its stack" when an exception is thrown. Basically it has to remove entries on its call stack, until a catcher is found.
Which is the process we already described [above](#rough).

So, inferring from the name again, we can guess that `_Unwind_Resume` will resume an interrupted stack unwinding (which is
[correct](https://web.archive.org/web/20201016174844id_/https://refspecs.linuxfoundation.org/LSB_4.0.0/LSB-Core-S390/LSB-Core-S390/baselib--unwind-resume.html),
by the way). And our cleanup function does interrupt the unwind, so after it returns, the program calls `_Unwind_Resume` to resume it.

Alright, _if_, we can stop that function from being called. If we can do that, we would have "caught" the exception.

Basically, we want `a_cleanup_function` to skip over everything in `might_throw` that is after the call to `a_cleanup_function`. Clearly, we need something that could interrupt the normal execution flow of the program.
`goto` is not enough, because our skip is cross-function. We need something stronger.

And there is indeed something stronger. `setjmp` and `longjmp`! A call to `longjmp` allows you to jump to a point in the program where you have previously called `setjmp` (the
[man page](https://man7.org/linux/man-pages/man3/setjmp.3.html) explains these functions very well). So we just need to use `setjmp` to put an anchor in the normal return path of `might_throw`, then have
`a_cleanup_function` jump to there with `longjmp`, and we would have skipped the `_Unwind_Resume`.

Here is a version of `might_throw` that does this:

```c
#include <setjmp.h>
struct Catch {
	jmp_buf env;

	// The cleanup function will be called when the exception is thrown,
	// but then it will be called again when we return from `might_throw`.
	// We don't want it to perform the longjmp when `might_throw` is returning
	// normally, so we use a flag to indicate whether a longjmp should be
	// performed.
	int do_jump;
};
void a_cleanup_function(struct Catch *env) {
	if (env->do_jump) {
		longjmp(env->env, 1);
	}
}
void might_throw() {
	__attribute__((cleanup(a_cleanup_function))) struct Catch env;
	if (setjmp(env.env) == 0) {
		env.do_jump = 1;
		throw_exception();
	} else {
		fprintf(stderr, "exception caught\n");
		env.do_jump = 0;
	}
	fprintf(stderr, "do more stuff after having caught the exception\n");
}
```

And running it, we will get:

```plain
exception caught
do more stuff after having caught the exception
```

Voilà! We have caught a C++ exception in C.

OK, are we done?

No! This result might be good enough, but I am still not satisfied. We have successfully _stopped_ an exception in C code, but we don't actually _get_ the thing that was thrown.
All we know in our "exception handler" is an exception was thrown, but not what the exception is. This is quite useless.

So, can we retrieve the thrown exception?

_Maybe_. We will continue in part 2.

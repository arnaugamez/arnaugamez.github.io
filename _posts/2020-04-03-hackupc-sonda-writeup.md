---
title: "Solving sonda reversing challenge with radare2"
comments: true

categories:
  - Blog
tags:
  - thegame
  - sonda
  - hackupc
  - radare2
  - esil
  - emulation
  - reversing
  - r2pipe

toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
toc_sticky: true

header:
  teaser: /assets/images/posts/sonda_bb_main_3.png
---

This was the reversing challenge from the HackUPC *TheGame* CTF-like competition. A slightly modified version has also been used recently during the *Advent of Corona* running platform of challenges during the 2020 covid19 confinement.

## Description

We are encouraged to download a file named [sonda](/assets/files/posts/sonda) referred as SONDA.EXE, so it seems it will be an executable file. We can easily observe that the file is a typical linux executable using file utility

```
$ file sonda
sonda: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=5d07808273b712ed8fa47fa8205e11d6926784b3, not stripped
```

If we run it we are asked for a magic number. We can try to throw some random values just to observe that it shows a "BAD..." message and exits the program.

```
$ ./sonda
Give me the magic number: 1234
BAD...
```

It is clear that first thing we will need is to get the correct magic number value.

Let's fire up our beloved [radare2](https://github.com/radareorg/radare2) and start analysing it.

> **Disclaimer**: although I have included some explanations, if you are new to radare2 and/or want a review of capabilities and some more practical usage, you might want to take a look at the materials (video/slides/github) from my 2h introduction to radare2 from Hack In The Box 2019. Check it out under [talks](/talks) section.

## Basic analysis

We open sonda file with r2 and use `iq` command to get quick **i**nformation about the file in **q**uiet (reduced) form.

```
$ r2 sonda
[0x000007c0]> iq
arch x86
bits 64
os linux
endian little
minopsz 1
maxopsz 16
pcalign 0
```

We can see that radare2 reports that we are dealing with a x86-64 linux (ELF64) executable, as we already knew. If you are interested you can explore different options for information extraction with in-line help of **i**nformation command using `i?`.

Now we will check strings on the binary using `iz`.


```
[0x000007c0]> iz
[Strings]
Num Paddr      Vaddr      Len Size Section  Type  String
000 0x00000bd4 0x00000bd4  26  27 (.rodata) ascii Give me the magic number: 
001 0x00000bf2 0x00000bf2   6   7 (.rodata) ascii BAD...
002 0x00000bf9 0x00000bf9  14  15 (.rodata) ascii Tell me more: 
003 0x00000c0b 0x00000c0b  20  21 (.rodata) ascii WTF is wrong with u?
004 0x00000c20 0x00000c20  20  21 (.rodata) ascii NOOB! Keep trying...
005 0x00000c35 0x00000c35   9  10 (.rodata) ascii flag{\%s}\n
```

Just from here without even running the executable, we can guess the flow of the binary: it looks like it will ask for a "magic number". If we provide the correct input, then will ask us for more input, make some checking again and, if correct, will print us the flag we are looking for.

Let's now use `aaa` (**a**nalyse **a**ll **a**utoname) to let radare2 make some of its magic with automatic analysis of the binary (check `a?` and `aa?` for more information). Now we can list the functions found with `afl` (**a**nalysis **f**unctions **l**ist)

```
[0x000007c0]> afl
0x000007c0    1 42           entry0
0x000007f0    4 50   -> 40   sym.deregister_tm_clones
0x00000830    4 66   -> 57   sym.register_tm_clones
0x00000880    5 58   -> 51   entry.fini0
0x000008c0    1 10           entry.init0
0x00000bc0    1 2            sym.__libc_csu_fini
0x00000bc4    1 9            sym._fini
0x00000b50    4 101          sym.__libc_csu_init
0x000008ca   20 642          main
0x000006f0    3 23           sym._init
0x00000720    1 6            sym.imp.free
0x00000730    1 6            sym.imp.puts
0x00000740    1 6            sym.imp.strlen
0x00000750    1 6            sym.imp.__stack_chk_fail
0x00000760    1 6            sym.imp.printf
0x00000000    2 25           loc.imp._ITM_deregisterTMCloneTable
0x00000770    1 6            sym.imp.srand
0x00000780    1 6            sym.imp.malloc
0x00000790    1 6            sym.imp.__isoc99_scanf
0x000007a0    1 6            sym.imp.rand
```

We can see a bunch of different functions, including some familiar imported ones. However, it looks like the more interesting (and biggest) is the main function, so we will start digging in there.

Now the funny part starts. Let's disassemble the main function. We can do it in many different ways, but the most useful in this case would probably be to check the graph view of the function. This is done with `VV` command, and we can specify a temporary seek for the main address with `@` (otherwise we would have to **s**eek at main offset with `s main` before using `VV`).

```
[0x000007c0]> VV @ main
```

Once in the graph view, we can repeatedly press `p` in order to change the amount/form of information displayed on it. If we just press one time it will show us the memory address for every instruction, which will come handy.

Here we have the big picture of the main function as a graph view of its basic blocks content and interaction, that we will be exploring by parts.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/sonda_graph_zoom_out.png"><img src="/assets/images/posts/sonda_graph_zoom_out.png"></a>
    <figcaption>Graph view of main function (zoomed out)</figcaption>
</figure>

Note that while we are on graph view, we should use colon `:` to be able to input r2 commands, as happens with the visual modes that radare2 offers.

## Obtaining the magic number

### Understand the problem

If we observe the flow at the end of first basic block of main function we can see that the input value we provided through the first scanf (remember, the "magic number") is copied from `ecx` into `eax` at `0x92a` (it was copied before into `ecx` after returning from scanf call) and it is compared to the value in `edx` that will contain the result of the previous operations.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/sonda_bb_main_1.png"><img src="/assets/images/posts/sonda_bb_main_1.png"></a>
    <figcaption>Graph view of main first basic blocks</figcaption>
</figure>


The comparison is done by subtracting `edx` from `eax` on `0x92c` and then using the `test eax, eax` instruction, which will make an `AND` logic operation on `eax` and set zero flag if result is zero. That basically means that zero flag will be set only if `eax` (which contains our input value) and `edx` contained the same value before the subtraction.

So we need to obtain the value that gets into `edx` for the comparison but... not that fast. Observe that the input value (the magic number) will be used for the computations that will affect the value at the end stored at `edx` so we can't just debug it, place a breakpoint before comparison, give a random input number and retrieve value at `edx`. We can proceed in two different ways:

- Statically decode/understand the process that manipulates input value, mixes and plays with it in different ways and stores value in `edx`

- Bruteforce it

First approach would be fairly easy in this case. Indeed, if you know just a little of x86 assembly and how usual and simple code constructs map to it, you probably already recognize this sequence. But for today, let's explore the second option of bruteforcing it, motivated by two reasons:

- Show you how we can leverage radare2's code emulation with ESIL by using the bruteforce approach as an excuse for a practical example.
- If you observe the basic block that follows the false branch of the comparison (in this case as we have a `jne` the false branch will actually be taken when the zero flag was set, i.e. when `eax` and `edx` had same value as we are looking for) you will see that it makes an extra check on the input value to be _lower or equal_ than `0x14` in order to continue to the meaningful part of the program and not to show bad message and exit. This basically means that the input magic number will be at most `0x14 = 20`, so bruteforce should go very quick.

### Emulation idea

The basic idea for bruteforce the needed input value with emulation will be as follows (note that emulation options are under **ae** (sub)commands, standing for **a**nalysis **e**sil. Check them with `ae?`):

1. Initialize ESIL VM state with `aei`
2. Set `ecx` to 0 (in the context of the emulation engine VM; remember we are not actually running the program!) with `aer ecx = 0`
3. Seek to `0x90e` (where the snippet of code computing the desired value starts) with `s 0x90e`
4. Set instruction pointer for ESIL to current seeked position with `aeip`
5. Emulate execution (stepping) until `0x92c` (just before the subtraction is done) with `aesu 0x92c`
6. Compare values of `eax` (gets loaded from `ecx` where our input is stored) and `edx`
7. If they are equal, we found a valid number input
8. Increment `ecx`
9. Repeat from 2 until `ecx` is less or equal than `0x14 = 20`

### Scripting with r2pipe to perform the bruteforce

We will use r2pipe API to script on top of r2 with python and do the bruteforce more easily, as it is extremely simple to use yet very powerful. As a super quick introduction, it is worth mentioning that r2pipe API consists of 4 methods:

- ***open***
- ***cmd***: input a r2 command and get r2 output as output
- ***cmdj***: input a r2 command with ***j*** suffix that returns json output, it will deserialize into native object.
- ***quit***

With that in mind we can create this simple script that will iterate over possible values and print when there is a possible candidate:

```python
import r2pipe
r2 = r2.open("./sonda")

r2.cmd("aei")

for i in range(0x14 + 1)
    r2.cmd("aer ecx = " + i)
    r2.cmd("s 0x90e")
    r2.cmd("aeip")
    r2.cmd("aesu 0x92c")
    if (r2.cmd("aer eax") == r2.cmd("aer edx")):
        print("Candidate magic number: " + i)
        
r2.quit()
```

We can run this script and get:

```
$ python3 brute_magic.py
Candidate magic number: 0
Candidate magic number: 17
```

As we can see we have two possible candidates. If you look at the basic block we arrive for valid magic number input (in the screenshot above), you can see that it uses that value as the size for a *malloc* call, so it is reasonable to make the educated guess that the actual **correct** magic number is intended to be 17, as a malloc of size 0 would not make much sense.

By the way, the operation being performed is exactly a *mod 17*, so basically is checking if `magic_number % 17 == 0`. That explains why 0 is valid as well. Also you might notice that other multiples of 17 would work as well for the first check, but they won't pass the second check to be less than 20.

> If you are interested in how emulation with ESIL is implemented and works, you might want to take a look at my talk about emulation with ESIL at past r2con2019. Check it out under [talks](/talks) section.

## Obtaining the actual flag

If we run the program and use as magic number the value 17 we just got, we can observe that it asks for another input.

```
$ ./sonda 
Give me the magic number: 17
Tell me more: asdf
NOOB! Keep trying...
```

### Flag information

Let's continue by checking the code where we land after the first check for magic number is successfully passed:

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/sonda_bb_main_2.png"><img src="/assets/images/posts/sonda_bb_main_2.png"></a>
    <figcaption>Graph view of main basic block after correct magic number</figcaption>
</figure>


We can see that the false branch will be taken if the result of a call to the *strlen* imported function, using the second input that we are asked as argument, returns a value greater than the magic number we introduced, leading us to another angry message and exiting the program. So this basically means that the second input has to have length at most 17. You can try to throw more than 17 chars on second input and will get the WTF message:

```
$ ./sonda 
Give me the magic number: 17
Tell me more: AAAAAAAAAAAAAAAAAAAAAAAAA
WTF is wrong with u?
```

Another important thing to note is that the pointer returned by the malloc in this basic block is the same as the pointer that will be accessed to print its value on the basic block that will display the flag value (near the end of main function). Note that the pointer has been automatically renamed to `ptr` by radare2 magic applied on the analysis:

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/sonda_bb_main_flag.png"><img src="/assets/images/posts/sonda_bb_main_flag.png"></a>
    <figcaption>Graph view of main basic block that will print flag</figcaption>
</figure>


What this is telling us is that the string we will need to use as input now is going to be the actual flag, so only when we reconstruct the flag content, we will be able to use it as input and therefore the basic block that will print us back the flag will be reached.

You might have noticed that there is only one path to the basic block printing the flag. Indeed, it will only be reached if the value stored at `var_34h` is less than our first input value, the magic number (observe that radare2 was smart enough to automatically rename it to *size* when we performed the initial analysis).

We will now explore the last few basic blocks before exiting. Those will be the meaningful ones to be able to discover the appropriate input to get to the flag, as the previous ones after the second input are just setting up some values that will be used later in here.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/sonda_bb_main_3.png"><img src="/assets/images/posts/sonda_bb_main_3.png"></a>
    <figcaption>Graph view of main basic blocks reaching end of function</figcaption>
</figure>

Notice that the `var_34h` that was used before for the check into flag's basic block is indeed a counter, as can be seen in the one-line basic block at the right. It is already safe to assume (and it is this way indeed, if you take a closer look to the code above) that the correct flag will be somehow checking char by char against our second input. The actual comparison is actually happening on `0xac8`.  If the char is correct, then it will increase the `var_34h` counter and proceed with next char.

To be more precise, observe that what is being compared is not actually the current char value, but the accumulated sum of the current char value with the previous ones. I encourage you to take a closer look by yourself navigating through this part of the code and trying to understand this loop construction of comparing the accumulated sum element by element, increasing a counter until some fixed value, as it is quite common and you will encounter it continuously in you reversing adventures.

### Retrieve flag with debugging

Our strategy to obtain the flag will be based on debugging the binary and retrieve each value individually in the moment of the comparison. To do so, we will first need to open the binary on debug mode with `-d` flag. This will of course change the addresses we view, as starting now we will be looking at addresses of process memory, instead of the memory layout of the file in disk which starts at offset 0.

We will use a [rarun2](https://radare.gitbooks.io/radare2book/tools/rarun2/intro.html) directive in order to disable [aslr](https://en.wikipedia.org/wiki/Address_space_layout_randomization) for the process, so we can reuse explicit address references if needed. Note that we can inline rarun2 directives when spawning the r2 debug shell with `-R` flag, instead of creating a rarun2 file. Thus, to open the binary in debug mode with specified rarun2 directive we do:

```
$ r2 -d -R aslr=no sonda
```

Please note that the [base address](https://en.wikipedia.org/wiki/Base_address) being used for the debugging session of the binary is `0x555555554000` by default, as aslr has been disabled. You can check its value with `ob`.

Now the procedure will be as follows (note that **d**ebug options are under **d** (sub)commands. Check them with `d?`):

1. Locate the comparison described before. It will be trivial to found by just following the main function's graph.
2. Patch conditional jump `je` after comparison for an unconditional jump `jmp`  that will always take us to next loop cycle despite our input value being wrong (so the comparison condition failing) with `wa jmp 0x555555554af8 @ 0x555555554acb` (**w**rite **a**ssembly ASM at ADDRESS).
3. Place a **b**reakpoint in that address with `db 0x555555554ac8`.
4. **C**ontinue program until it breaks with `dc`.
5. Get `eax` **r**egister value. This can be done with `dr rax`.
6. Repeat steps 4 and 5 until flag message is printed (it will be after 17 steps, as is is controlled by the counter described previously that gets compared to the first input value).

Instead of patching the jump after the comparison, we could have followed some other strategies that would have led us to same results, for example patching instruction pointer `rip` with value as if comparison had been successful with `dr rip = 0x555555554af8` every time after we reach the breakpoint.

Feel free to experiment this other alternative and any other you could think of. The result will still be the same, and we will be able to retrieve the flag.

Please note that as we are on a debug session, when we are applying those memory patches, they are applied into the process memory, but no modification is made to the file in disk.

For practical reasons we will combine steps 4 and 5 by just using a semicolon between them `dc; dr rax`. Indeed, we can combine any number of r2 commands in a single line by just separating them with a semicolon.

Moreover, we will define a [macro](https://radare.gitbooks.io/radare2book/scripting/macros.html) that will make exactly that, so we can call it directly and any number of times. A macro can be defined as `(macro_name; cmd1; cmd2; cmd3 ...)` and then called with `.(macro_name)`. We can prefix it with the number of times `N` to be called, so it will be `N.(macro_name)`

Let's take a look at what we get after all the steps described:

```
[0x7ffff7fd1100]> wa jmp 0x555555554af8 @ 0x555555554acb
Written 2 byte(s) (jmp 0x555555554af8) = wx eb2b
[0x7ffff7fd1100]> db 0x555555554ac8
[0x7ffff7fd1100]> (loop; dc; dr rax)
[0x7ffff7fd1100]> 17.(loop)
Give me the magic number: 17
Tell me more: asdf
hit breakpoint at: 555555554ac8
0x00000036
hit breakpoint at: 555555554ac8
0x000000a4
hit breakpoint at: 555555554ac8
0x00000120
hit breakpoint at: 555555554ac8
0x0000016c
hit breakpoint at: 555555554ac8
0x0000019c
hit breakpoint at: 555555554ac8
0x000001f2
hit breakpoint at: 555555554ac8
0x00000214
hit breakpoint at: 555555554ac8
0x0000024a
hit breakpoint at: 555555554ac8
0x00000288
hit breakpoint at: 555555554ac8
0x000002ee
hit breakpoint at: 555555554ac8
0x0000034a
hit breakpoint at: 555555554ac8
0x0000036e
hit breakpoint at: 555555554ac8
0x000003b8
hit breakpoint at: 555555554ac8
0x000003fd
hit breakpoint at: 555555554ac8
0x00000478
hit breakpoint at: 555555554ac8
0x000004ed
hit breakpoint at: 555555554ac8
0x00000546
```

We can check that we reached the last comparison by continuing program with `dc` and observe that it will print us the flag message. Of course it won't be the correct flag, as it will only load our input and display it wrapped inside `flag{}`. You might wonder why is the flag message being printed at all. Basically, as we patched the jump instruction after comparison to be always treated as if it was successful, the program will act as if the flag was correct so it will print us back our input, which, as we explained before, will actually the flag content that we need to reconstruct.

```
[0x555555554ac8]> dc
flag{asdf}
```

Now we have all the values of `rax` at each step, which will contain the accumulated sum of the chars for the actual flag. So starting from 2nd value, we basically need to subtract the previous value from the current and we will get the flag char by char:

- `0x36 = 54 = '6'`
- `0xa4 - 0x36 = 110 = 'n'`
- `0x120 - 0xa4 = 124 = '|'`
- ...

So boring to do it manually, right? Of course you can get the output and parse with your favorite shell utilities and/or scripting language to make it for you. We will demonstrate again the use of the r2pipe python API to automate this process by directly manipulating the r2 output for `rax` at each step.

### Automate flag extraction with r2pipe

In order to do it completely automatic, we will define another rarun2 directive that will specify a file that will serve to feed the stdin of the program. We create a plain text file that will contain the following to feed both inputs:

```
$ cat input.txt
17
asdf

```

That is the magic number 17 and then the second input, which can be totally random (provided it is at most 17 chars long so it does not throw us to the WTF message), as all the checks on it will be bypassed by the jump patch. Those will be passed when program asks for input from stdin through scanf function calls.

As we are now scripting with python, we don't need to define a macro at all. Indeed, it will be better just to iterate 17 times and store the `rax` value and subtract it from previous one to get the correct char of the flag at each step. Afterwards we will simply print the obtained flag.

```python
import r2pipe
r2 = r2pipe.open("./sonda", flags=['-d', '-R', 'stdin=input.txt', '-R', 'aslr=no'])

flag = ""
pre = 0

r2.cmd("wa jmp 0x555555554af8 @ 0x555555554acb")
r2.cmd("db 0x555555554ac8")

for i in range(17):
    r2.cmd("dc")
    cur = int(r2.cmd("dr rax"), 16)
    flag += chr(cur - pre)
    pre = cur

print("\n---\nFlag obtained -> " + flag)
```

Let's run this script and chill while it does all the work for us.

```
$ python3 solve_sonda.py
Process with PID 806 started...
= attach 806 806
bin.baddr 0x555555554000
Using 0x555555554000
asm.bits 64
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8
hit breakpoint at: 555555554ac8

---
Flag obtained -> 6n|L0V"6>f\$JE{uY
```

Eureka! Here we have our beloved flag. Let's try it to verify it.

```
$ ./sonda
Give me the magic number: 17
Tell me more: 6n|L0V"6>f\$JE{uY
flag{6n|L0V"6>f\$JE{uY}
```

So, that will be it. We obtained the solution `flag{6n|L0V"6>f\$JE{uY}`

## Some final *random* thoughts

You might wonder why the flag is so *weird*. I wondered too, so I took a deeper look.

We previously skipped a couple of basic blocks that were not essential to investigate, as all the information needed to retrieve the flag during the comparison was already prepared when reaching it. If you take a closer look at those, you will notice that the flag generation depends on the outcome of the random number generator (RNG) that was initialized using the *magic number* input as its seed.

It is safe to assume that the author of the challenge just decided that the flag would be of size 17, then initialized the RNG with this seed and observed the outcome of the performed mangling on top of the values thrown by this RNG and directly used that as a flag. It would have been quite tedious (not impossible at all, maybe just not worth or time-effective) to create a more personalized flag while relying on the RNG output.

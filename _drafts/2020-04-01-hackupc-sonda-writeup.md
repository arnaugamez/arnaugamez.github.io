---
title: "[HackUPC2019 - TheGame] Solving sonda reversing challenge with radare2"
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

toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
toc_sticky: true

---



## Description

We are encouraged to download a file named [sonda](/assets/files/posts/sonda) referred as SONDA.EXE so it seems it will be an executable file.

If we run it we are asked for a magic number. We can try to throw some random values just to observe that it shows a "BAD..." message and exits the program.

```
$ ./sonda
Give me the magic number: 1234
BAD...
```

It is clear that first thing we will need is to get the correct magic number value.

Let's fire up our beloved [radare2](https://github.com/radareorg/radare2) and start analysing it.

> **Disclaimer**: although I might include some brief explanations, this is not intended to be a tutorial on how does radare2 work. If you are new to radare2 you might want to take a look at the materials (video/slides/github) from my 2h introduction to radare2 from Hack In The Box 2019. Check it out under [talks](/talks) section.

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

As we can see, we are dealing with a x86-64 linux (ELF64) executable. If you are interested you can explore different options for information extraction with in-line help of information command using `i?`.

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

We can see a bunch of imported functions, but looks like the more interesting (and biggest) is the main function, so we will start digging in there.

Now the funny part starts. Let's disassemble the main function. We can do it in many different ways, but the most useful in this case would be to check the graph view of the function. This is done with `VV` command, and we can specify a temporary seek for the main address with `@` (otherwise we would have to **s**eek at main offset with `s main` before using `VV`).

Once in the graph view, we can repeatedly press `p` in order to change the amount/form of information displayed on it. If we just press one time it will show us the memory address for every instruction, which will come handy.

Here we have the big picture of the main function as a graph view, that we will be exploring by parts afterwards.

<figure class="align-center" style="width:385px">
    <a href="/assets/images/posts/sonda_graph_zoom_out.png"><img src="/assets/images/posts/sonda_graph_zoom_out.png"></a>
    <figcaption>Graph view of main function (zoomed out)</figcaption>
</figure>
## Obtaining the magic number

### Understand the problem

If we observe the flow at the end of first basic block of main function we can see that the input value we provided through the first scanf (remember, the "magic number") is copied into eax at `0x92a` (it was before copied into ecx) and it is compared to the value in `edx` that will contain the result of the previous operations.

<figure class="align-center" style="width:385px">
    <a href="/assets/images/posts/sonda_bb_main_1.png"><img src="/assets/images/posts/sonda_bb_main_1.png"></a>
    <figcaption>Graph view of main first basic blocks</figcaption>
</figure>

The comparison is done by subtracting edx from eax on `0x92c` and then using the `test eax, eax` instruction, which will make an `AND` logic operation into eax and set zero flag if result is zero. That basically means that zero flag will be set only if `eax` (which contains our input value) and `edx` have the same value.

So we need to obtain the value that gets into `edx` for the comparison but... not that fast. Observe that the input value will be used for the computations that will affect the value at the end stored at `edx` so we can't just debug it, place a breakpoint before comparison, give a random input number and retrieve value at `edx`. We can proceed in two different ways:

- Statically decode/understand the process that manipulates input value, mixes and plays with it in different ways and stores value in `edx`

- Bruteforce it

First approach would be fairly easy in this case. Indeed, if you know just a little of x86 and how usual and simple code constructs map to it, you already recognize this sequence, but for today, let's explore the second option of bruteforcing it, motivated by two reasons:

- Show you how we can leverage code emulation with ESIL using the bruteforce approach as an excuse for a practical example.
- If you observe the basic block that follows the false branch of the comparison (in this case as we have a `jne` the false branch will actually be when the zero flag was set, i.e. when `eax` and `edx` had same value as we are looking for) you will see that if makes an extra check on the input value to _lower or equal_ than 0x14 in order to continue to the meaningful part of the program and not go to show bad message and exit. This basically means that the input magic number will be at most 0x14, so bruteforce should go very quick.

### Emulation idea

The basic idea for bruteforce the needed input value with emulation will be as follows (note that emulation options are under **ae** (sub)commands, standing for **a**nalysis **e**sil):

1. Initialize ESIL VM state with `aei`
2. Set `ecx` to 0 (in the context of the emulation engine VM; we are not actually running the program!) with `aer ecx = 0`
3. Seek to 0x90e where the snipped of code computing the desired value starts with `s 0x90e`
4. Set instruction pointer for ESIL to current seeked position with `aeip`
5. Emulate (stepping) until 0x92c before the subtraction is done with `aesu 0x92c`
6. Compare values of `eax` (gets loaded from `ecx` where our input is stored) and `edx`
7. Id they are equal, we found a valid number input
8. Increment `ecx`
9. Repeat from 2 until `ecx` is less or equal than 0x14 = 20

### Scripting with r2pipe to perform the bruteforce

We will use r2pipe API to script on top of r2 with python and do the bruteforce more easily, as it is extremely simple to use yet very powerful. r2pipe API consists of 4 methods:

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

As we can see we have two possible candidates. If you look at the basic block we arrive for valid magic number input, you can see that it uses that value as the size for a *malloc* call, so it is reasonable to make the educated guess that the actual **correct** magic number is intended to be 17, as a malloc of size 0 would not make much sense.

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

Let's check the code where we land after the first check for magic number is successfully passed:

<figure class="align-center" style="width:385px">
    <a href="/assets/images/posts/sonda_bb_main_2.png"><img src="/assets/images/posts/sonda_bb_main_2.png"></a>
    <figcaption>Graph view of main basic block after correct magic number</figcaption>
</figure>

We can see that the false branch will be taken if the result of a call to the *strlen* imported function, using the second input that we are asked as argument, returns a value greater than the magic number we introduced, leading us to another angry message and will exit the program. So this basically means that the second input has to have length at most 17. You can try to throw more than 17 chars on second input and will get the WTF message:

```
$ ./sonda 
Give me the magic number: 17
Tell me more: AAAAAAAAAAAAAAAAAAAAAAAAA
WTF is wrong with u?
```

Another important thing to note is that the pointer returned by the malloc in this basic block is the same as the pointer that will be accessed to print its value on the basic block that will display the flag value (near the end of main function):

<figure class="align-center" style="width:385px">
    <a href="/assets/images/posts/sonda_bb_main_flag.png"><img src="/assets/images/posts/sonda_bb_main_flag.png"></a>
    <figcaption>Graph view of main basic block that will print flag</figcaption>
</figure>

What this is telling us is that the string we will need to use as input now is going to be the actual flag, so only when we reconstruct the flag content, we will be able to use it as input and therefore the basic block that will print us back the flag will be reached.

You might have noticed that there is only one path to the basic block printing the flag. It will only be reached if the value stored at `var_34h` is less than our input value (observe that radare2 was smart enough to automatically rename it to *size* when we performed the initial analysis).

After some digging and skipping some messing that we are not interested in, we can see the last few basic blocks before exiting.

<figure class="align-center" style="width:385px">
    <a href="/assets/images/posts/sonda_bb_main_3.png"><img src="/assets/images/posts/sonda_bb_main_3.png"></a>
    <figcaption>Graph view of main basic blocks reaching end of function</figcaption>
</figure>

Notice that the `var_34h` that was used before for the check into flag's basic block is indeed a counter, as can be seen in the one-line basic block at the right. It is already safe to assume (and it is this way indeed, if you take a closer look to the code above) that the correct flag will be checking char by char against our second input. The actual char comparison is actually happening on `0xac8`.  If the char is correct, then it will increase the `var_34h` counter and proceed with next char.

[EXPLAIN ACCUMULATIVE CHARS]

I encourage you to take a closer look by yourself navigating through the code and trying to understand this loop construction of comparing element by element, increasing a counter until some fixed value, as it is quite common and you will encounter it continuously in you reversing adventures.

### Retrieve flag with debugging

Our strategy to obtain the flag will be based on debugging the binary and retrieve each char individually. To do so, we will first need to open the binary on debug mode with `-d` flag. This will of course change the addresses we view, as now we are looking at addresses of process memory, instead of the memory layout of the file in disk (starting at offset 0).

We will use a [rarun2](https://radare.gitbooks.io/radare2book/tools/rarun2/intro.html) directive in order to disable aslr for the process, so we can reuse address references if needed. Note that we can inline rarun2 directives when spawning the r2 debug shell with `-R` flag, instead of creating a rarun2 file. Thus, to open the binary in debug mode with specified rarun2 directive we do:

```
$ r2 -d -R aslr=no sonda
```

Now the procedure will be as follows (note that **d**ebug options are under **d** (sub)commands):

1. Locate the comparison described before. It will be trivial to found by just following the main function's graph.
2. Place a breakpoint in that address with `db address`
3. Continue program until it breaks with `dc`
4. Get `eax` value. This can be done with `dr rax` [...]
5. Patch instruction pointer `rip` with value as if comparison has been successful with `dr rip = 0x555555554af8`
6. Repeat from 2 until flag message is printed (it will be after 17 steps, as is is controlled by the counter described previously that gets compared to the first input value).



Now we have the cumulative [...]



### Automate flag extraction with r2pipe

[...]

The input.txt file will contain the following:

```
$ cat input.txt
17
AAAAAAAAAAAAAAAAA

```

That is the magic number 17 and then this number of A's. Those will be passed when program asks for input from stdin through scanf function calls.

[...]

```python
import r2pipe

flag = ""
pre = 0

r2 = r2pipe.open("./sonda", flags=['-d', '-R', 'stdin=input.txt', '-R', 'aslr=no'])
r2.cmd("db 0x555555554ac8")

for i in range(17):
    r2.cmd("dc")
    cur = int(r2.cmd("dr rax"), 16)
    flag += chr(cur - pre)
    pre = cur
    r2.cmd("dr rip = 0x555555554af8")

print("Flag obtained -> " + flag)
```

[...]

```
$ python solve_sonda.py
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
Flag obtained -> 6n|L0V"6>f\$JE{uY
```

[...]

```
$ ./sonda
Give me the magic number: 17
Tell me more: 6n|L0V"6>f\$JE{uY
flag{6n|L0V"6>f\$JE{uY}
```

[...]
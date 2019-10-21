---
title: "[HackUPC2019 - TheGame] Solving sonda reversing challenge with radare2"
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
---

## Description

We are encouraged to download a file named [sonda](/assets/files/posts/sonda) refered as SONDA.EXE so it seems it will be an executable file.

Let's fire up our beloved [radare2](https://github.com/radareorg/radare2) and start analysing it.

> **Disclaimer**: although I might include some brief explanations, this is not intended to be a tutorial on how does radare2 work. If you are new to radare2 you might want to take a look at the materials (video/slides/github) from my 2h introduction to radare2 from Hack In The Box 2019. Check it out under [talks](/talks) section.

## Analysis

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

Let's now use `aa` (**a**nalyse **a**ll) to let radare2 make some of its magic with automatic analysis of the binary (check `a?` and `aa?` for more information). Now we can list the functions found with `afl` (**a**nalysis **f**unctions **l**ist)

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

Now the funny part starts. Let's disassemble the main function. We can do it in many different ways, but the most useful in this case would be to check the graph view of the function. This is done with `VV` command, and we can specify a temporary seek for the main address with `@` (otherwise we would have to seek at main offset with `s main` before using `VV`).

Once in the graph view, we can repeatedly press `p` in order to change the amount/form of information displayed on it. If we just press one time it will show us the memory address for every instruction, which will come handy.

Here we have the big picture of the main function as a graph view, that we will be exploring by parts afterwards.

<figure class="align-center" style="width:385px">
    <a href="/assets/images/posts/sonda_graph_zoom_out.png"><img src="/assets/images/posts/sonda_graph_zoom_out.png"></a>
    <figcaption>Graph view of main function (zoomed out)</figcaption>
</figure>



## Obtaining the magic number




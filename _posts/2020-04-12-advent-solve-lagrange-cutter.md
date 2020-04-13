---
title: "Solving lagrange reversing challenge with Cutter"
categories:
  - Blog
tags:
  - radare2
  - reversing
  - Cutter
  - hackupc
  - advent

toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
toc_sticky: true

header:
  teaser: /assets/images/posts/lagrange_cutter_6.png
---

The Lagrange reversing challenge was the 6th problem included in the [*Advent of Corona*](https://adventofcorona.hackersatupc.org) CTF platform. The platform was uploading challenges during the first weeks of covid19 confinement, and it is still up if you want to take a look and play.



## Description

We download the binary file [lagrange_baby](/assets/files/posts/lagrange_baby) from the platform, which includes the following statement for the challenge:

> Lagrange would be proud, maybe even Newton...

We can check that it is, indeed, a 64-bit Linux ELF executable:

```
$ file lagrange_baby
lagrange_baby: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=57b67fb4328e4f7989c7453333bb77c112105f0f, not stripped
```

If we run it, looks like it just waits for an input. We provide some random input to observe the outcome:

```
$ ./lagrange_baby
1234
noup
```

```
$ ./lagrange_baby
asdf
noup
```

This time we will be using [Cutter](https://cutter.re), the official radare2 GUI. Let's load our file into it and select default options for analysis:

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_1.png"><img src="/assets/images/posts/lagrange_cutter_1.png"></a>
    <figcaption>Load binary on cutter with default analysis options</figcaption>
</figure>

It will take no time to analyse it and prompt us with its pretty UI (notice that the specific windows layout can be different from your default one. You can adapt it to your needs and preferences).

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_2.png"><img src="/assets/images/posts/lagrange_cutter_2.png"></a>
    <figcaption>Cutter main view</figcaption>
</figure>

## Obtaining first input

### Strings and references

Let's take a look at the strings on the .rodata section to see if they offer some hints:

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_3.png"><img src="/assets/images/posts/lagrange_cutter_3.png"></a>
    <figcaption>Strings on .rodata section</figcaption>
</figure>

From the strings found, we can assume that it will make three different checks on our input (or inputs). If all of them succeed it will print the correct flag, which does not appear to be explicitly hard-coded.

The first string found is the one printed back to us when we tried some random inputs previously. We now select it and press `X` to show cross-references to it.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_4.png"><img src="/assets/images/posts/lagrange_cutter_4.png"></a>
    <figcaption>Cross references to "noup" string</figcaption>
</figure>

Lucky us, there is only one reference to this string. To show it on the graph view, let's do `Right click -> Show in -> Graph (main)`

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_5.png"><img src="/assets/images/posts/lagrange_cutter_5.png"></a>
    <figcaption>"noup" appearance in Graph view on main function</figcaption>
</figure>

### Understand initial loop (1st check)

Observe in the graph overview that we reach this block if the comparison in the previous block was unsuccessful. The comparison on this previous block depends on computations done in the previous three basic blocks that are in a row. You can easily observe that there is a loop involving those three basic blocks, and the outcome of this loop will be checked (presumably against our input) to proceed to next check, or to exit passing through the "noup" printing message.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_6.png"><img src="/assets/images/posts/lagrange_cutter_6.png"></a>
    <figcaption>Graph overview on main function</figcaption>
</figure>

To understand this loop we will take advantage of the integrated Ghidra decompiler. First we will fix some mismatching types between ghidra-dec and r2 analysis to avoid variable overlapping and so the decompiler output will be a little bit cleaner. Also we will change their names to describe more precisely their meaning. To do so we just need to select the desired variable on the disassembly listing on the graph view itself and press `Y`.

- Change `size` variable to `int32_t`. Also change its name to `input_1` as this will hold the value from the first input.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_7.png"><img src="/assets/images/posts/lagrange_cutter_7.png"></a>
    <figcaption>Change size varible type and name</figcaption>
</figure>

- Change `var_2ch` to `uint32_t` . Also change its name to `counter_1`.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_8.png"><img src="/assets/images/posts/lagrange_cutter_8.png"></a>
    <figcaption>Change var_2ch varible type and name</figcaption>
</figure>

After those changes, we can observe that the described loop maps to the following pseudocode obtained from the decompiler.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_9.png"><img src="/assets/images/posts/lagrange_cutter_9.png"></a>
    <figcaption>Decompilation of first loop</figcaption>
</figure>

### Bruteforce first input

Observe that we included the check that is made just after the loop. This is the check that will lead us to the "noup" message if it fails. As we can see, for the check to be successful, we need our input to be equal to the `iVar3` that results from the previous loop computations. Let's just bruteforce its value by reproducing the loop on a quick python script.

Note that the challenge author has been kind enough to keep symbols, so we can just assume that the `sym.isprime` function does exactly what one expects it to do; checking it the parameter passed is a prime number.

```python
from sympy import isprime

counter_1 = 0xd #13

while (True):
  if (isprime(counter_1) and isprime(counter_1 % 10)):
    iVar3 = counter_1 // 10
    if (iVar3 % 3 == 0):
      if ((iVar3 - (iVar3 >> 0x1f) & 1) + (iVar3 >> 0x1f) == 0):
        print(iVar3)
        break
      
  counter_1 += 1
```

If we run it, it will give us the solution to be the number `6`. Therefore, we have now the correct value for the first required input.

## Obtaining second input (flag)

### Length of second input (2nd check)

Now we can take a look at the coming flow after the first check has been passed successfully.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_10.png"><img src="/assets/images/posts/lagrange_cutter_10.png"></a>
    <figcaption>Second input and check</figcaption>
</figure>

As you can see, it makes a `malloc` of the number of bytes corresponding to the first input value. So it will be a malloc of 6 bytes. Then is asks for a second input with the call to `scanf` function, storing the value into the memory address pointed by string pointer `ptr`. After that, it checks the length of this second input to be equal as the first input number `6`.  If the check fails, it will go to print us another error message "still noup" (remember it from the strings listing we have seen before).

### Understand last loop (3rd check)

If we provide a as second input a string of length 6 it will then go through the last part of the main function that can be seen here:

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_11.png"><img src="/assets/images/posts/lagrange_cutter_11.png"></a>
    <figcaption>Last part of the main function</figcaption>
</figure>

It is obvious that the `var_28h` is another counter for a loop. It gets initialized with 0. Then it is compared against the first input, which, as we have just seen, it should be the length of the second input string, i.e. `6`. While the value of `var_28h` is less than `6` it will perform some computations and another comparison. If the comparison succeeds, its value will  be incremented by 1 and loop back to the comparison against first input.

(Feel free to rename the `var_28h` variable to `counter_2` for example. This can be done as before: select it and press `Y`)

If the check within this loop fails at any iteration, the program will go to print us the "aaaaaaaaand still noup" string and exit. Therefore, it is clear that all checks must be passed (a total of 6) and then the flag will be printed to us.

Observe that the pointer that is passed for the flag printing is `ptr`. This is the same pointer to our second input string. This fact reveals that our second input should actually be the correct flag.

Let's take a look into the basic block responsible for the comparison in the loop.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_12.png"><img src="/assets/images/posts/lagrange_cutter_12.png"></a>
    <figcaption>Loop comparison for flag</figcaption>
</figure>

It is clear that it compares each byte (char) from our second input string to the result obtained from calling the `sym.epic_function`. We don't really need to dig into this function, as we are only interested in its output at each iteration. This output will essentially represent the *i-th* character of the correct flag.

### Debug to get the flag char by char

We will now start a Cutter debugging session (*yay! we already have debugging support within Cutter :D*) by clicking the "Play" icon in the top bar. This will take us to the debugging view as we can see here:

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/lagrange_cutter_12.5.png"><img src="/assets/images/posts/lagrange_cutter_12.5.png"></a>
    <figcaption>Debugging view on Cutter</figcaption>
</figure>

The process to get the flag at each iteration will be as follows:

1. Place a breakpoint in the comparison instruction within the loop checking the flag. This can be done selecting the line `cmp ebx, eax` and pressing `F2` or clicking on it and going to `Breakpoint -> Add breakpoint`.

2. Patch the conditional jump in the next instruction to be inconditional so it will always continue the loop. this can be done by clicking into it and then `Edit -> Instruction`.

   <figure class="align-center" style="width:300px">
       <a href="/assets/images/posts/lagrange_cutter_13.png"><img src="/assets/images/posts/lagrange_cutter_13.png"></a>
       <figcaption>Change instruction to inconditional jump</figcaption>
   </figure>

3. Continue until program breaks. This can be done by pressing `F5` or clicking in the "fast forward" icon in the top bar. You will need to provide the inputs. This can be easily done within Cutter directly. Just go to the bottom-left and change the "R2 Console" value of the menu into "Debugee Input". Then provide appropriate inputs in order. First `6` and then any 6-byte-long string like `AAAAAA` so it will pass the second check directly. After providing the inputs, the program will break.

   <figure class="align-center" style="width:300px">
       <a href="/assets/images/posts/lagrange_cutter_14.png"><img src="/assets/images/posts/lagrange_cutter_14.png"></a>
       <figcaption>Provide input within the debugging session</figcaption>
   </figure>

4. Check value at `eax`, it will contain the *i-th* character of the correct flag string. We can see it in the register's window (under the `rax` register, as `eax` is essentially the lower 32-bit part of the 64-bit register `rax`). A really quick and simple way to get the ASCII representation is just to hover over it.

   <figure class="align-center" style="width:300px">
       <a href="/assets/images/posts/lagrange_cutter_15.png"><img src="/assets/images/posts/lagrange_cutter_15.png"></a>
       <figcaption>Check correct char of flag</figcaption>
   </figure>

5. Repeat until it exits.

After 6 iterations, we will have the correct flag `C0roN4`

We can now test it running the program and providing the correct inputs we obtained:

```
$ ./lagrange_baby 
6
C0roN4
flag{C0roN4}
```

As expected, the program returned `flag{C0roN4}` as output, so we have successfully solved the challenge.

## Additional comments

This challenge is a good example of two important things to keep in mind while reversing:

- We can take a lot of advantage if we *somehow* have symbols for the binary. In this case they were provided embedded as the binary was non stripped. This allowed us to super quickly assume that the `sym.isprime` function did what it looked like it had to do. I encourage you to take a look at the function itself and imagine having to conduct the same reversing session but not knowing its symbol name. You will probably find quite lost inside a function that that relies on random procedures and making more nested function calls. It is not trivial at all to discover that this function is *just* checking if a given number is prime. If you are curious, it is actually implementing the [Miller-Rabin primalty test](https://en.wikipedia.org/wiki/Miller%E2%80%93Rabin_primality_test).
- It is very easy to get lost into details that are non important to our objective. In this binary we can take as example the `sym.epic_function`. If you take a look into it, you will see that it makes some floating point operations as well as some calls to another function called `sym.pow_mod`. You could spend a pretty large amount of time dealing with those trying to understand what is actually happening inside them, and probably getting frustrated as well. However, as you have seen above, there is no need to dig into them, as we are only interested in its plain outcome, regardless of its inner implementation. Anyway, for the curious people out there, I am pretty sure that the `sym.epic_function` is just a hard-coded polynomial that has been constructed ad-hoc by interpolating the pairs of inputs and outputs for each char of the flag string. That is, for I/O pairs (`1 + counter_2 at i-th iteration`, `i-th char of flag string`). Feel free to spend some time reversing and reconstructing it.
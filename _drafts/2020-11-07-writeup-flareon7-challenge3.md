---
title: "Write-up for FlareOn7 challenge #3 - wednesday"
comments: true

categories:
  - Blog
tags:
  - flareon
  - reversing
  - CTF

toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
toc_sticky: true

header:
  teaser: /assets/images/posts/flareon7_ch2/upx_bad.png
---

## Description
We are introduced to the challenge with the following message:

> Be the wednesday. Unlike challenge 1, you probably won't be able to beat this game the old fashioned way. Read the README.txt file, it is very important. 

If we open the `README.txt` file, we find the following:

>                        --- BE THE WEDNESDAY ---
> 
>                                    S
>                                    M
>                                    T
>                                   DUDE
>                                    T
>                                    F
>                                    S
> 
>                 --- Enable accelerated graphics in VM ---
>                   --- Attach sound card device to VM ---
>                     --- Only reverse mydude.exe ---
>                        --- Enjoy it my dudes ---



We set up our VM accordingly and start analyzing the file `mydude.exe` as indicated.

## Analysis
### Basic exploration
If we load our binary into [Detect It Easy](http://ntinfo.biz/index.html), we find that `mydude.exe` is a 32-bit PE Windows executable.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/die.png"><img src="/assets/images/posts/flareon7_ch3/die.png"></a>
    <figcaption>Output of Detect It Easy (DIE)</figcaption>
</figure>

We launch `mydude.exe` and are prompted with what appears to be the welcome screen of a weird game called `Wednesday` starring an amorphous frog.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/welcome_screen_game.png"><img src="/assets/images/posts/flareon7_ch3/welcome_screen_game.png"></a>
    <figcaption>Wednesday's welcome screen</figcaption>
</figure>

Clicking into the `DUDE` button will get us into the actual game screen. It appears to be a simple game where we have to avoid the obstacles in order to advance, either by jumping over them or ducking down. If we hit one of the obstacles, the game is automatically reset.

Moreover, it looks like we do not only need to avoid the obstacles, but also we must do so in a certain fashion for each obstacle (according to its appearing order), either by jumping or ducking.


<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/game.gif"><img src="/assets/images/posts/flareon7_ch3/game.gif"></a>
    <figcaption>Wednesday's game screen</figcaption>
</figure>

After a couple minutes playing, we start to discover the pattern that describes the way in which each appearing obstacle has to be dodged. In particular, for the first obstacles, we have:

```
_ _ * * _ _ _ * _ * * * _ * _ _ ... jump (*) | duck (_)
```

### Locate interesting parts of code

Let's load `mydude.exe` in IDA, and make sure to allow it to load the embedded symbols detected. A ton of functions will be found, most of them having seemingly weird/random suffixes. Such rare function names appear to be related to the fact that the game has been coded in [Nim](https://nim-lang.org/) (you can find a bunch of symbols containing "Nim" in it).

Although we have symbols, we still have a huge amount of functions and code to explore. Thus, in order to locate interesting sections of the code logic, we will use a common trick from the old days of game hacking: finding interesting values in memory for _things_ whose values change in a predictable and known manner. By locating them, we will be able to track back to meaningul code that references them. In this case, we will use [Cheat Engine](https://www.cheatengine.org) to do so.

One obvious candidate is the `Score` value that gets incremented any time we pass an obstacle. We just need to start the game (`Score = 0`) and inmediately look for memory locations storing the value `0` with `New Scan` and a `Value: 0`. After we pass a couple obstacles, we can quickly change the value field to `Value: 2` and clicking `Next Scan`. This way, we will find memory locations that held a value of `0` during first scan, and a value of `2` during second scan.


<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/cheat_engine_1.png"><img src="/assets/images/posts/flareon7_ch3/cheat_engine_1.png"></a>
    <figcaption>Cheat Engine showing the two memory references to Score value</figcaption>
</figure>

We find two interesting memory locations, `0x00443D64` and `0x0044DDB0`, lying within our executable memory region and referencing values having the exact behavior of the score being updated as we pass through obstacles. These memory addresses have been named `SCORE_REF_1` and `SCORE_REF_2` in Cheat Engine for clarity purposes.

Let's take a look at the memory address `0x00443D64` in IDA. We find that it is a named symbol starting with `_prev_score`. Also, it gets cross referenced by functions `@resetEverything__Q1G0...` and `@update__Arw3...`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/prev_score_ref.png"><img src="/assets/images/posts/flareon7_ch3/prev_score_ref.png"></a>
    <figcaption>IDA's view of first reference to Score value found</figcaption>
</figure>

If we take a quick look into the `@resetEverything__Q1G0...` function, we instantly find in its first basic block a reference to an interesting memory location named `_obstacles__Xqz...`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/resetEverything_fun_obstacles_highlight.png"><img src="/assets/images/posts/flareon7_ch3/resetEverything_fun_obstacles_highlight.png"></a>
    <figcaption>First basic block of @resetEverything__Q1G0... function</figcaption>
</figure>

At its turn, `obstacles__Xqz...` points to another memory location named `_TM__V45tF8...`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/obstacles_array.png"><img src="/assets/images/posts/flareon7_ch3/obstacles_array.png"></a>
    <figcaption>Memory view of _TM__V45tF8... value pointed by obstacles__Xqz...</figcaption>
</figure>

But wait... we observe a <<table>> of `1`'s and `0`'s describing a familiar pattern:

```
_ _ * * _ _ _ * _ * * * _ * _ _
0 0 1 1 0 0 0 1 0 1 1 1 0 1 0 0
```

Exactly! We find the same pattern we encountered while playing the game at the very beginning. If we take a closer look, there are `296` (`0x00000128`) values, either `1` or `0`, conforming this array of bytes.

Let's take a look at cross references to `_obstacles_Xqz...`. We find three functions reading it. From these, one appears to be an initialization function and another one is the `@resetEverything__Q1G0...` function where we found it being referenced before.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/xrefs_to_obstacles.png"><img src="/assets/images/posts/flareon7_ch3/xrefs_to_obstacles.png"></a>
    <figcaption>Cross references to _obstacles_Xqz...</figcaption>
</figure>

Notice that the third function is `@update__Arw3...`, the same function that appeared also as a cross reference to the `_prev_score` memory location, so it is a good idea to dig deeper into this function.

But, before exploring this `@update__Arw3...` function, let's take a look at `0x0044DDB0`, which is the other interesting memory location we found on Cheat Engine. We find that it is a named symbol starting with `_score` and cross referenced by the function `@onCollide__9byA...`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/score_ref.png"><img src="/assets/images/posts/flareon7_ch3/score_ref.png"></a>
    <figcaption>IDA's view of second reference to Score value found</figcaption>
</figure>

If we pull the thread on cross references, we find that `@onCollide__9byA...` is referenced by `@onCollide__BN6X...`, which is referenced by `@checkCollisions__P9bT...`, which is referenced by `@updateScene__rbzI...`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/xrefs_to_1st_oncollide.png"><img src="/assets/images/posts/flareon7_ch3/xrefs_to_1st_oncollide.png"></a>
    <figcaption>Cross references to @onCollide__9byA...</figcaption>
</figure>

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/xrefs_to_2nd_oncollide.png"><img src="/assets/images/posts/flareon7_ch3/xrefs_to_2nd_oncollide.png"></a>
    <figcaption>Cross references to @onCollide__BN6X...</figcaption>
</figure>

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/xrefs_to_checkCollisions.png"><img src="/assets/images/posts/flareon7_ch3/xrefs_to_checkCollisions.png"></a>
    <figcaption>Cross references to @checkCollisions__P9bT...</figcaption>
</figure>

Now let's finally explore `@update__Arw3...` function. We find that it calls `@updateScene__rbzI...`, followed by a conditional jump that will lead into a call to `@resetEverything__Q1G0...` in case that the byte value pointed by the memory location stored at `eax+F9` is equal to 1.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/update_fun.png"><img src="/assets/images/posts/flareon7_ch3/update_fun.png"></a>
    <figcaption>First basic blocks of @update__Arw3... function</figcaption>
</figure>

So, with the information we have obtained so far, we can make an educated guess about the inner working of the game with respect to collision management:
- The game gets constantly updated (for example, frame based updates)
- Every time the game updates, it will check for <<general collisions>> with an obstacle.
- Every time the game goes through an obstacle, it will check for <<table collisions>>, i.e. whether the way in which the obstacle is dodged coincides with the correspoinding move, as described on the obstacle's table.
- If either of the two collisions occur, the memory location pointed previously by `eax+F9` will be set to `1`. Thus, at next update, it will enter into the reset function to restart the game.
- After `296` obstacles successfully passed, <<something>> would happen; presumably, the flag should be revealed in some way. 


## Bypass collisions
Our objective will be to patch the binary so that we can bypass all kinds of collisions while preserving the game behavior as if we completed it in a _legit_ way.

The hasty idea would be to just patch the conditional jump after the comparison `cmp byte ptr [eax+0F9h], 1` so it never gets into the basic block calling the reset function. However, this does not look like a good idea, as the code logic for recreating and showing the flag could (and indeed, will) be dependant of the actual code flow around the checks that set the memory location pointed by `eax+F9` to `1` at that point.

Thus, our approach will be to find where this memory location gets written to `1`, so we can explore the code around and patch effectively without altering the global code flow. To do so, we will use the [x64dbg](https://x64dbg.com/) debugger (well, actually the 32-bit version, x32dbg).

We load `mydude.exe` on x32dbg and let it run until it shows the welcome screen. Now we place a regular breakpoint at `0x433D55` (the location of the previously described comparison) and click on `DUDE` to start the game.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/x64dbg_bp1.png"><img src="/assets/images/posts/flareon7_ch3/x64dbg_bp1.png"></a>
    <figcaption>Breakpoint set at 0x433D55 in x32dbg</figcaption>
</figure>

The breakpoint will be hit inmediately and pause the execution. Now, let's get the memory address pointed by `eax+F9` into the memory dump view by doing: `Right click -> Follow in Dump -> Address: EAX+F9`

Then, we will add a hardware breakpoint on write access (at byte level, as the comparison is done at byte level as well) to this memory location. This can be seen in the following screenshot.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/x64dbg_bp2.png"><img src="/assets/images/posts/flareon7_ch3/x64dbg_bp2.png"></a>
    <figcaption>Set hardware breakpoint on write access to the byte pointed by eax+F9</figcaption>
</figure>

We can now remove (or disable) the previous breakpoint on the comparison. Let's continue running the program (pressing F9) and observe that the hardware breakpoint is getting hit repeatedly at two different places.
- At `0x43266C` the memory location gets written to `0`.
- At `0x432247` the memory location gets written to `1`.

It looks like these writes are used at each game update to control if a <<general>> collision happened.

We are only interested in the location where it gets set to `1` (indicating a collision). Thus, in order not to hit the breakpoint for the location that sets the value to `0`, we will modify the hardware breakpoint to be conditional. In particular, we set it not to be triggered whenever the `EIP` would land into the address `0x432673`, which is the instruction following the write instruction at `0x43266C`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/edit_hardware_bp.png"><img src="/assets/images/posts/flareon7_ch3/edit_hardware_bp.png"></a>
    <figcaption>Edit hardware breakpoint to add condition</figcaption>
</figure>

### Patch general collision check

If we take a look at the code flow around this write at `0x432247`, we observe that we arrive to the basic block containing it depending on a previous conditional jump `js`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/general_collision_check.png"><img src="/assets/images/posts/flareon7_ch3/general_collision_check.png"></a>
    <figcaption>Code flow around general collision write at 0x432247</figcaption>
</figure>

One quick and naive approach would be to patch this `1` (indicating collision) into a `0`. However, this would not guarantee that the correct code path is followed. Thus, it is safer to patch the conditional jump to always take the correct path, where the memory location of our interest would not be written to `1`.

We can easily patch this conditional jump into an inconditional jump by modifying the assembly instruction in the debugger, from `js` into `jmp`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/patch_general_collisions.png"><img src="/assets/images/posts/flareon7_ch3/patch_general_collisions.png"></a>
    <figcaption>Patch conditional jump to bypass general collision check</figcaption>
</figure>


### Patch table collision check
Now, we allow the game to run, noticing that it does not break instantly as before (the `0` write is not triggered because of the condition in the hardware breakpoint; the `1` write is bypassed  with the previous patch to the conditional jump).

Indeed, we observe that if we start the game, it will go over the first two obstacles without hitting them and will only trigger the hardware breakpoint when hitting the third obstacle. This makes sense, as this third obstacle is the first one in which we would have to jump over, i.e. we have the first `1` at third position in the pattern/table we found before.

This time, the breakpoint will lead us to a write of value `1` to the memory location we are tracking, ocurring at `0x43235E`. 

If we take a look at the code flow around this write at `0x43235E`, we observe  a similar situation as before: we arrive to the basic block containing the write depending on a previous conditional jump `jz`.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/table_collision_check.png"><img src="/assets/images/posts/flareon7_ch3/table_collision_check.png"></a>
    <figcaption>Code flow around table collision write at 0x43235E</figcaption>
</figure>

As we did before, instead of patching this `1` into a `0`, we will patch the conditional jump by changing this `jz` into `jmp` in the debugger, so it always follows the correct path.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/patch_table_collisions.png"><img src="/assets/images/posts/flareon7_ch3/patch_table_collisions.png"></a>
    <figcaption>Patch conditional jump to bypass table collision check</figcaption>
</figure>

## Retrieve flag

We have patched both conditionals that led to basic blocks containing writes of value `1` (indicating collision) into the memory location that was checked for restarting the game in case of a collision. Essentially, we have bypassed both <<general>> and <<table>> collisions, while preserving as much as possible the control flow followed by the original code.

Thus, now it is just a matter of letting it run for a while. After 296 obstacles passed, it will show us a victory screen with the string "Winner!" as well as our flag: `1t_i5_wEdn3sd4y_mY_Dud3s@flare-on.com`


<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch3/flag.png"><img src="/assets/images/posts/flareon7_ch3/flag.png"></a>
    <figcaption>Wednesday's victory screen showing the flag</figcaption>
</figure>

## A quicker alternative

Actually, if we had taken the table of `296` `1`'s and `0`'s and treated them as binary digits, we could have decoded them and obtained the flag directly without the need to allow the game to be run.

As an example, a simple python decoding routine is shown below.

```python
table = "00110001011101000101111101101001001101010101111101110111010001010110010001101110001100110111001101100100001101000111100101011111011011010101100101011111010001000111010101100100001100110111001101000000011001100110110001100001011100100110010100101101011011110110111000101110011000110110111101101101"
n = int(table, 2)
flag = n.to_bytes((n.bit_length() + 7) // 8, 'big').decode('ascii')
print (flag)
```

```
$ python dec.py
1t_i5_wEdn3sd4y_mY_Dud3s@flare-on.com
```

This approach might have saved some time, but also would have spoiled the fun we had during our journey with this fancy frog ;)

Anyway, we obtained the flag and completed the third challenge!
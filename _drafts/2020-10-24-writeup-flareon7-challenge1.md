---
title: "Write-up for FlareOn7 challenge #1 - fidler"
comments: true

categories:
  - Blog
tags:
  - flareon
  - reversing
  - python
  - CTF

toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
toc_sticky: true
---

*
GENERIC HEADER WITH LINKS TO ALL OTHER WRITE-UPS
*


## Description
We are introduced to the challenge with the following message:

> Welcome to the Seventh Flare-On Challenge!
>
> This is a simple game. Win it by any means necessary and the victory screen will reveal the flag. Enter the flag here on this site to score and move on to the next level.
>
> This challenge is written in Python and is distributed as a runnable EXE and matching source code for your convenience. You can run the source code directly on any Python platform with PyGame if you would prefer.


I did not even bother to install PyGame and went straight into the only interesting file `fidler.py`, which contains all the logic of the program.

## Analysis

It is a small and very easy to follow python program. Within a quick glimpse, we inmediately bump into the following two functions: `decode_flag` and `victory_screen`.


```python
def decode_flag(frob):
    last_value = frob
    encoded_flag = [1135, 1038, 1126, 1028, 1117, 1071, 1094, 1077, 1121, 1087, 1110, 1092, 1072, 1095, 1090, 1027,
                    1127, 1040, 1137, 1030, 1127, 1099, 1062, 1101, 1123, 1027, 1136, 1054]
    decoded_flag = []

    for i in range(len(encoded_flag)):
        c = encoded_flag[i]
        val = (c - ((i%2)*1 + (i%3)*2)) ^ last_value
        decoded_flag.append(val)
        last_value = c

    return ''.join([chr(x) for x in decoded_flag])


def victory_screen(token):
    screen = pg.display.set_mode((640, 160))
    clock = pg.time.Clock()
    heading = Label(20, 20, 'If the following key ends with @flare-on.com you probably won!',
                    color=pg.Color('gold'), font=pg.font.Font('fonts/arial.ttf', 22))
    flag_label = Label(20, 105, 'Flag:', color=pg.Color('gold'), font=pg.font.Font('fonts/arial.ttf', 22))
    flag_content_label = Label(120, 100, 'the_flag_goes_here',
                               color=pg.Color('red'), font=pg.font.Font('fonts/arial.ttf', 32))

    controls = [heading, flag_label, flag_content_label]
    done = False

    flag_content_label.change_text(decode_flag(token))
    
    # --snip--
```

The function `decode_flag` runs a fairly easy decryption routine to retrive the flag, depending on the parameter `frob` received. This function will be called from `victory_screen` function passing as argument the same `token` parameter it received, without applying any modification to it.

If we look for calls to `victory_screen` function, we find the following snippet of code under `game_screen` function:

```python
def game_screen():
    # --snip--
    target_amount = (2**36) + (2**35)
    if current_coins > (target_amount - 2**20):
        while current_coins >= (target_amount + 2**20):
            current_coins -= 2**20
        victory_screen(int(current_coins / 10**8))
        return
    # --snip--
```

We see that the `token` argument passed to `victory_screen` will be equal to `int(current_coins / 10**8)`. Thus, we need to obtain which value(s) of `current_coints` will satisfy the required conditions to arrive to this call to `victory_screen`.

## Obtain token value

Observe that to enter the body of the first `if`, we need that:
```python
current_coins > 2^36 + 2^35 - 2^20 = 103078166528
```
Then, it will enter a `while` loop that decreases the value of `current_coins` by `2^20` until the following condition is met:
```python
current_coins < 2^36 + 2^35 + 2^20 = 103080263680
```
Thus, we get that `current_coins` must satisfy:
```python
103078166528 < current_coins < 103080263680
```

Notice that `current_coins` is then divided by `10^8` and casted to `int` type. This will essentially remove the 8 least significant digits of `current_coins`, meaning that any `current_coins` value within the valid interval will lead to the `token` argument having a value of `1030`. Thus, our token must be `1030`.


## Obtain flag

To get the flag, we can simply extract the `decode_flag` function into a new python file and print the value it returns when passing `1030` as argument.

```python
def decode_flag(frob):
    last_value = frob
    encoded_flag = [1135, 1038, 1126, 1028, 1117, 1071, 1094, 1077, 1121, 1087, 1110, 1092, 1072, 1095, 1090, 1027,
                    1127, 1040, 1137, 1030, 1127, 1099, 1062, 1101, 1123, 1027, 1136, 1054]
    decoded_flag = []

    for i in range(len(encoded_flag)):
        c = encoded_flag[i]
        val = (c - ((i%2)*1 + (i%3)*2)) ^ last_value
        decoded_flag.append(val)
        last_value = c

    return ''.join([chr(x) for x in decoded_flag])

print(decode_flag(1030))
```

```
$ python decoder.py
idle_with_kitty@flare-on.com
```

So, we obtained the flag `idle_with_kitty@flare-on.com` and can move onto the next challenge!

*
LINK TO CHALLENGE 2 POST.
*
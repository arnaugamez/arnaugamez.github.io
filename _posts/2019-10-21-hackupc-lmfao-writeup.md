---
title: "Solving LMFAO stego challenge"
comments: true

categories:
  - Blog
tags:
  - hackupc
  - writeup
  - stego
  - steganography
  - thegame

toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
toc_sticky: true

header:
  teaser: /assets/images/posts/LMFAO.png

---

Last week I attended the [HackUPC](https://hackupc.com/) at Barcelona. They were running a 24h CTF-like competition with different challenges focused on algorithms, programming and some hacking. I wasn't planning to play, but some friends got me involved in the last moment and I somehow managed to win the competition by completing all challenges and getting max score, along with Yuva.

![The Game - Hall of Fame](/assets/images/posts/TheGame_HOF.jpg){: .align-center}

Here you will find my writeup for the *LMFAO* steganography challenge.

## Description

We are just provided the following image and random footer:

<figure class="align-center" style="width: 300px">
    <a href="/assets/images/posts/LMFAO.png"><img src="/assets/images/posts/LMFAO.png"></a>
    <figcaption>I like eggs for breakfast</figcaption>
</figure>

It seems pretty clear that this image has been scrambled somehow from an original one. Thus, our goal will be to recover the original image.

We will need some information before starting to randomly move pixels around.

## Analysis

Let's start by applying some filters to the image in order to see if we can find something suspicious. For this, we will use [Stegsolve.jar](http://www.caesum.com/handbook/Stegsolve.jar) utility.

<figure class="align-center" style="width:300px">
    <a href="/assets/images/posts/LMFAO_stegsolve_1.gif"><img src="/assets/images/posts/LMFAO_stegsolve_1.gif"></a>
    <figcaption>Apply filters with Stegsolve</figcaption>
</figure>

Observe that the filters corresponding to masking with bit 0 (Least Significant Bit) for the three color channels, clearly show that something is encoded on the upper part of the image in those bits. We can explore it and extract the data with Stegsolve itself.

Going to `Analyse -> Data Extract`we can play with its values. We are interested in bit 0 from RGB channels. If we select those bits and with default Order settings, we will find the following data:

<figure class="align-center">
    <a href="/assets/images/posts/LMFAO_stegsolve_2.png"><img src="/assets/images/posts/LMFAO_stegsolve_2.png"></a>
    <figcaption>Extract data with Stegsolve (w/o hexdump)</figcaption>
</figure>

Now it's obvious that this data is indicating us how the image was scrambled. We assume V stands for Vertical (columns) and H for Horizontal (rows). Also the following two numbers meaning the rows/columns swapped. Therefore, we have that:

`V 236 39` means swapping columns 236 and 39

`H 262 7` means swapping rows 262 and 7

and so on...

Finally we will save the data as text clicking `Save Text` button to get a textfile ready to be loaded.

## Recover original image

Now that we know how the image was scrambled, it is just a matter of transforming what we have into the original image by scrambling back the image that we are given, i.e. applying the swapping operations found in reverse order.

We can do this easily with python. Let's see how:

### Read image and extract color channels

We will use the `io` subpackage from `skimage` library in order to read the image:

```python
from skimage import io
im = io.imread("LMFAO.png")
```

Then we will need to get each color channel in order to apply the transformations to all of them afterwards

```python
r = im[:,:,0]
g = im[:,:,1]
b = im[:,:,2]
```

### Read and parse data extracted previously

Now we will read the file `moves.txt` that we saved before, containing the extracted data, and will create a list containing all the required information with the form: `['V', '236', '39', 'H', '262', '7', ...]`

```python
f = open("moves.txt", 'r')
moves = ''.join([x[:8]+x[9:-1] for x in f.readlines()[:581]])[:-15].split()
```

In order to get a clean list from the input data as we wanted, we needed to slightly massage it:

`x[:8]+x[9:-1]`gets rid of a space on position 8th of each line, as well as removing newline `\n` character.

`[:581]` gets rid of lines 582 ongoing, as they do not contain more data

`[:-15]` removes last 15 characters from obtained string, as the last line (n. 581) has only one meaningful character (8, at the very beginning)

### Apply transformations in reverse order

We will traverse the generated list from end to beginning and apply the transformations in that order. Let's see the code to do this:

```python
l = len(moves) - 1
for i in range(0, l, 3):
    p1 = int(moves[l - i - 1])
    p2 = int(moves[l - i])

    if moves[l - i - 2] == "H":
        r[[p1, p2], :] = r[[p2, p1], :]
        g[[p1, p2], :] = g[[p2, p1], :]
        b[[p1, p2], :] = b[[p2, p1], :]
    
    if moves[l - i - 2] == "V":
        r[:, [p1, p2]] = r[:, [p2, p1]]
        g[:, [p1, p2]] = g[:, [p2, p1]]
        b[:, [p1, p2]] = b[:, [p2, p1]]
```

We simply assign `p1` to the first operand and `p2` to the second one. Then if we find an `H` will mean that it is an horizontal swap, i.e. swapping rows, and if we find a `V` will mean that it is a vertical swap, i.e. swapping columns. Performing the transformation is easily achieved just by list indexing as shown in code.

### Putting all together

After all, we get to the following complete code:

```python
from skimage import io

# Read image and extract color channels
im = io.imread("LMFAO.png")
r = im[:,:,0]
g = im[:,:,1]
b = im[:,:,2]

# Read and parse txt output from Stegsolve
f = open("moves.txt", 'r')
moves = ''.join([x[:8]+x[9:-1] for x in f.readlines()[:581]])[:-15].split()

# Apply transformations in reverse order
l = len(moves) - 1
for i in range(0, l, 3):
    p1 = int(moves[l - i - 1])
    p2 = int(moves[l - i])

    if moves[l - i - 2] == "H":
        r[[p1, p2], :] = r[[p2, p1], :]
        g[[p1, p2], :] = g[[p2, p1], :]
        b[[p1, p2], :] = b[[p2, p1], :]
    
    if moves[l - i - 2] == "V":
        r[:, [p1, p2]] = r[:, [p2, p1]]
        g[:, [p1, p2]] = g[:, [p2, p1]]
        b[:, [p1, p2]] = b[:, [p2, p1]]

# Save resulting image
io.imsave("LMFAO_solved.png", im)
```

Observe that we have just added the final line in order to save the resulting image into an actual file. 

## Solve

Now if we just run the code `python lmfao.py` it will produce the resulting image file `LMFAO_solved.png` from where we can extract the flag:

<figure class="align-center" style="width: 300px">
    <a href="/assets/images/posts/LMFAO_solved.png"><img src="/assets/images/posts/LMFAO_solved.png"></a>
    <figcaption>Recovered original image</figcaption>
</figure>
As we can see, we get the solution`flag{se_biene_bien_duro}` 

[![Twitter Follow](https://img.shields.io/twitter/follow/arnaugamez?style=social)](https://twitter.com/intent/follow?screen_name=arnaugamez)






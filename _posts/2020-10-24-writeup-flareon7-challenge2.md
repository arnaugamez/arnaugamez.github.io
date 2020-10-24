---
title: "Write-up for FlareOn7 challenge #2 - garbage"
comments: true

categories:
  - Blog
tags:
  - flareon
  - reversing
  - radare2
  - emulation
  - esil
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

> One of our team members developed a Flare-On challenge but accidentally deleted it. We recovered it using extreme digital forensic techniques but it seems to be corrupted. We would fix it but we are too busy solving today's most important information security threats affecting our global economy. You should be able to get it working again, reverse engineer it, and acquire the flag.

This challenge consists of the single file `garbage.exe`, so we will jump directly into analyzing it.

## Analysis
If we load our binary into [Detect It Easy](http://ntinfo.biz/index.html), we find that `garbage.exe` is a 32-bit PE Windows executable, which appears to be packed with the popular [UPX](https://upx.github.io/) packer.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/die.png"><img src="/assets/images/posts/flareon7_ch2/die.png"></a>
    <figcaption>Output of Detect It Easy (DIE)</figcaption>
</figure>

Inmediately, we try to unpack the file so we can proceed with our analysis. However, when we try to unpack the file with `upx` utility, an error message is thrown.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/upx_bad.png"><img src="/assets/images/posts/flareon7_ch2/upx_bad.png"></a>
    <figcaption>Output of upx failed unpacking</figcaption>
</figure>

Remember the description message of the challenge? This might be (part of) the file's corruption it refered to. It is clear that we need to fix the binary somehow before we can unpack it successfully.

## Fix binary and unpack

Let's load `garbage.exe` into [PE-Bear](https://hshrzd.wordpress.com/pe-bear/) to investigate what could be wrong with it. We quickly find a a red-colored area under the `.rsrc` section within the `Section Hdrs` tab.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/pe_bear_rsrc.png"><img src="/assets/images/posts/flareon7_ch2/pe_bear_rsrc.png"></a>
    <figcaption>Output of PE-Bear showing the mismatching .rsrc section size</figcaption>
</figure>

As it indicates, there is a mismatch between the `.rsrc` section size reported in the PE header (`0x400`) and its actual size within the file (`0x124`). To fix it, we will extend the file a total of `(0x400 - 0x124)` bytes. This can be easily done with [radare2](https://github.com/radareorg/radare2). First of all, we make a copy of `garbage.exe` into `fixed_garbage.exe` so we can keep the original one, just in case.

```
$ cp garbage.exe fixed_garbage.exe
$ radare2 -w fixed_garbage.exe
[0x00418760]> r
40740
[0x00418760]> r 40740 + (0x400 - 0x124)
[0x00418760]> r
41472
```

We open `fixed_garbage.exe` with radare2 in write mode (`-w`) so we can change its contents, which will be automatically stored on disk. The `r` command shows us current file size (in bytes). If we specify a new size after it, we will effectively change file size to the indicated new size. We show the new size just to check it.

> Fun fact: we could have used the following one-liner to achieve the same result without the need to get the initial file size, as we can use output of radare2 commands as values, similarly as we do in a regular shell:
```
[0x00418760]> r `r` + (0x400 - 0x124)
```

If we try again to unpack `fixed_garbage.exe` with `upx`, we will get a successful outcome resulting in the unpacked executable `unpacked_garbage.exe` being created.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/upx_good.png"><img src="/assets/images/posts/flareon7_ch2/upx_good.png"></a>
    <figcaption>Output of upx successful unpacking</figcaption>
</figure>

However, as we simply extended its size, the unpacked version will certainly still be corrupted. Indeed, if we attempt to launch `unpacked_garbage.exe`, we (still) get the following error message:

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/error_open_unpacked.png"><img src="/assets/images/posts/flareon7_ch2/error_open_unpacked.png"></a>
    <figcaption>Error executing unpacked_garbage.exe</figcaption>
</figure>

The *orthodox* way of proceeding would be analyzing the PE file and completely fixing it to the point we could execute/debug the binary. However, as we are a bit lazy to do so, and we know some radare2's emulation magic, we might try our luck to see if that would be enough.

> If you are interested in radare2's emulation with ESIL, I have a couple public [talks](/talks) covering it.


## Retrieve flag with emulation

Let's open `unpacked_garbage.exe` with radare2, analyze it and print the dissassembly of its main function.

```
$ radare2 unpacked_garbage.exe
[0x00401473]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Check for vtables
[x] Type matching analysis for all functions (aaft)
[x] Propagate noreturn information
[x] Use -AA or aaaa to perform additional experimental analysis.
[0x00401473]> s main
[0x0040106b]> pdf
            ; CALL XREF from entry0 @ 0x4013e6
┌ 432: int main (int argc, char **argv, char **envp);
│           ; var LPDWORD lpNumberOfBytesWritten @ ebp-0x13c
│           ; var LPCSTR lpFileName @ ebp-0x138
--snip--
│           0x00401084      mov esi, str.nPTnaGLkIqdcQwvieFQKGcTGOTbfMjDNmvibfBDdFBhoPaBbtfQuuGWYomtqTFqvBSKdUMmciqKSGZaosWCSoZlcIlyQpOwkcAgw ; 0x4119f8 ; "nPTnaGLkIqdcQwvieFQKGcTGOTbfMjDNmvibfBDdFBhoPaBbtfQuuGWYomtqTFqvBSKdUMmciqKSGZaosWCSoZlcIlyQpOwkcAgw "
--snip--
│           0x004010b3      mov esi, str.KglPFOsQDxBPXmclOpmsdLDEPMRWbMDzwhDGOyqAkVMRvnBeIkpZIhFznwVylfjrkqprBPAdPuaiVoVugQAlyOQQtxBNsTdPZgDH ; 0x411a60 ; "KglPFOsQDxBPXmclOpmsdLDEPMRWbMDzwhDGOyqAkVMRvnBeIkpZIhFznwVylfjrkqprBPAdPuaiVoVugQAlyOQQtxBNsTdPZgDH "
--snip--
│           0x00401150      push ebx                    ; HANDLE hTemplateFile
│           0x00401151      push 0x80                   ; 128 ; DWORD dwFlagsAndAttributes
│           0x00401156      push 2                      ; 2 ; DWORD dwCreationDisposition
│           0x00401158      push ebx                    ; LPSECURITY_ATTRIBUTES lpSecurityAttributes
│           0x00401159      push 2                      ; 2 ; DWORD dwShareMode
│           0x0040115b      push 0x40000000             ; DWORD dwDesiredAccess
│           0x00401160      push dword [lpFileName]     ; LPCSTR lpFileName
│           0x00401166      call dword [sym.imp._CreateFileA] ; 0x40d00c ; HANDLE CreateFileA(LPCSTR lpFileName, DWORD dwDesiredAccess, DWORD dwShareMode, LPSECURITY_ATTRIBUTES lpSecurityAttributes, DWORD dwCreationDisposition, DWORD dwFlagsAndAttributes, HANDLE hTemplateFile)
│           0x0040116c      lea ecx, [lpFileName]
--snip--
│       │   0x0040119d      push ebx                    ; LPOVERLAPPED lpOverlapped
│       │   0x0040119e      lea eax, [lpNumberOfBytesWritten]
│       │   0x004011a4      push eax                    ; LPDWORD lpNumberOfBytesWritten
│       │   0x004011a5      push 0x3d                   ; '=' ; 61 ; DWORD nNumberOfBytesToWrite
│       │   0x004011a7      push dword [lpFileName]     ; LPCVOID lpBuffer
│       │   0x004011ad      push esi                    ; HANDLE hFile
│       │   0x004011ae      call dword [sym.imp._WriteFile] ; 0x40d004 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
--snip--
```

Notice how there are a couple of suspicous strings being loaded. After that, we reach a call into `CreateFile` at `0x00401166` followed by another call into `WriteFile` at `0x004011ae`. Thus, we will emulate the code of the main function to retrieve the parameters being passed into those two function calls.

First, we initialize the ESIL emulation engine along its virtual memory with `aeim`. Then we step until the offset where `CreateFile` gets called with `aesu 0x00401166`. Now we show the contents of the local variables at this precise moment with `afvd`.

```
[0x0040106b]> aeim
[0x0040106b]> aesu 0x00401166
[0x00401150]> afvd
--snip--
var lpFileName = 0x00177ec4 = 0x00177ec4 -> 0x00177fe4 "sink_the_tanker.vbs"
--snip--
```

We observe a reference to the file `sink_the_tanker.vbs`, which looks like would be the name of the file being created if we actually fixed and executed the unpacked binary.

As we are not interested in getting into the mess of actually emulating the `CreateFile` function, we can just skip it changing the instruction pointer `eip` to the offset that comes after the function call. Then, we proceed to emulate until the call into `WriteFile` and show the contents of the local variables again. 

```
[0x00401150]> aer eip=0x0040116c
[0x00401150]> aesu 0x004011ae
[0x0040119d]> afvd
--snip--
var lpFileName = 0x00177ec4 = 0x00177ec4 -> 0x00177fa4 "MsgBox("Congrats! Your key is: C0rruptGarbag3@flare-on.com")"
--snip--
```

We get a reference to a string containing an `MsgBox` function, presumably to be written into the previous `sink_the_tanker.vbs` file. Even better, the message inside this MsgBox contains our flag. Awesome!

Thus, we obtained the flag `C0rruptGarbag3@flare-on.com` and successfully finished this challenge!
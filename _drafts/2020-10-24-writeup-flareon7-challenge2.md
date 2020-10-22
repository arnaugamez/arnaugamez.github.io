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
---

*
GENERIC HEADER WITH LINKS TO ALL OTHER WRITE-UPS
*

## Description
We are introduced to the challenge with the following message:

> One of our team members developed a Flare-On challenge but accidentally deleted it. We recovered it using extreme digital forensic techniques but it seems to be corrupted. We would fix it but we are too busy solving today's most important information security threats affecting our global economy. You should be able to get it working again, reverse engineer it, and acquire the flag.


## Analysis

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/die.png"><img src="/assets/images/posts/flareon7_ch2/die.png"></a>
    <figcaption>Loop comparison for flag</figcaption>
</figure>

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/upx_bad.png"><img src="/assets/images/posts/flareon7_ch2/upx_bad.png"></a>
    <figcaption>Loop comparison for flag</figcaption>
</figure>

### Fix binary

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/pe_bear_rsrc.png"><img src="/assets/images/posts/flareon7_ch2/pe_bear_rsrc.png"></a>
    <figcaption>Loop comparison for flag</figcaption>
</figure>

```
$ radare2 -w fixed_garbage.exe
[0x00418760]> r
40740
[0x00418760]> r 40740 + (0x400 - 0x124)
[0x00418760]> r
41472
```


### Unpack

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch2/upx_good.png"><img src="/assets/images/posts/flareon7_ch2/upx_good.png"></a>
    <figcaption>Loop comparison for flag</figcaption>
</figure>

If we try to run it still does not work.

Fix PE or do ESIL magic!


## Retrieve flag with emulation

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
│           0x00401084      bef8194100     mov esi, str.nPTnaGLkIqdcQwvieFQKGcTGOTbfMjDNmvibfBDdFBhoPaBbtfQuuGWYomtqTFqvBSKdUMmciqKSGZaosWCSoZlcIlyQpOwkcAgw ; 0x4119f8 ; "nPTnaGLkIqdcQwvieFQKGcTGOTbfMjDNmvibfBDdFBhoPaBbtfQuuGWYomtqTFqvBSKdUMmciqKSGZaosWCSoZlcIlyQpOwkcAgw "
--snip--
│           0x004010b3      be601a4100     mov esi, str.KglPFOsQDxBPXmclOpmsdLDEPMRWbMDzwhDGOyqAkVMRvnBeIkpZIhFznwVylfjrkqprBPAdPuaiVoVugQAlyOQQtxBNsTdPZgDH ; 0x411a60 ; "KglPFOsQDxBPXmclOpmsdLDEPMRWbMDzwhDGOyqAkVMRvnBeIkpZIhFznwVylfjrkqprBPAdPuaiVoVugQAlyOQQtxBNsTdPZgDH "
--snip--
│           0x00401150      53             push ebx                    ; HANDLE hTemplateFile
│           0x00401151      6880000000     push 0x80                   ; 128 ; DWORD dwFlagsAndAttributes
│           0x00401156      6a02           push 2                      ; 2 ; DWORD dwCreationDisposition
│           0x00401158      53             push ebx                    ; LPSECURITY_ATTRIBUTES lpSecurityAttributes
│           0x00401159      6a02           push 2                      ; 2 ; DWORD dwShareMode
│           0x0040115b      6800000040     push 0x40000000             ; DWORD dwDesiredAccess
│           0x00401160      ffb5c8feffff   push dword [lpFileName]     ; LPCSTR lpFileName
│           0x00401166      ff150cd04000   call dword [sym.imp._CreateFileA] ; 0x40d00c ; HANDLE CreateFileA(LPCSTR lpFileName, DWORD dwDesiredAccess, DWORD dwShareMode, LPSECURITY_ATTRIBUTES lpSecurityAttributes, DWORD dwCreationDisposition, DWORD dwFlagsAndAttributes, HANDLE hTemplateFile)
│           0x0040116c      8d8dc8feffff   lea ecx, [lpFileName]
--snip--
│       │   0x0040119d      53             push ebx                    ; LPOVERLAPPED lpOverlapped
│       │   0x0040119e      8d85c4feffff   lea eax, [lpNumberOfBytesWritten]
│       │   0x004011a4      50             push eax                    ; LPDWORD lpNumberOfBytesWritten
│       │   0x004011a5      6a3d           push 0x3d                   ; '=' ; 61 ; DWORD nNumberOfBytesToWrite
│       │   0x004011a7      ffb5c8feffff   push dword [lpFileName]     ; LPCVOID lpBuffer
│       │   0x004011ad      56             push esi                    ; HANDLE hFile
│       │   0x004011ae      ff1504d04000   call dword [sym.imp._WriteFile] ; 0x40d004 ; BOOL WriteFile(HANDLE hFile, LPCVOID lpBuffer, DWORD nNumberOfBytesToWrite, LPDWORD lpNumberOfBytesWritten, LPOVERLAPPED lpOverlapped)
--snip--
```

```
[0x0040106b]> aeim
[0x0040106b]> aesu 0x00401166
[0x00401150]> aer eip=0x0040116c
[0x00401150]> aesu 0x004011ae
[0x0040119d]> afvd
--snip--
var lpFileName = 0x00177ec4 = 0x00177ec4 -> 0x00177fa4 "MsgBox("Congrats! Your key is: C0rruptGarbag3@flare-on.com")"
--snip--
```

We obtained the flag `C0rruptGarbag3@flare-on.com` and successfully finished this challenge!

*
LINK TO CHALLENGE 3 POST.
*
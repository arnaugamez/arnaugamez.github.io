---
title: "Write-up for FlareOn7 challenge #4 - report"
comments: true

categories:
  - Blog
tags:
  - flareon
  - reversing
  - CTF
  - macros
  - excel

toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
toc_sticky: true

header:
  teaser: /assets/images/posts/flareon7_ch4/bypass_protections.png
---

## Description
We are introduced to the challenge with the following message:

> Nobody likes analysing infected documents, but it pays the bills. Reverse this macro thrill-ride to discover how to get it to show you the key.

There is a single file `report.xls` to analyze, which is an excel document with the old windows format. From the challenge description and the kind of file we are given, we can be sure that we will have to deal with an excel macro.

## Analysis
If we open the document we find the following image within the spreadsheet.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch4/report_initial_image.png"><img src="/assets/images/posts/flareon7_ch4/report_initial_image.png"></a>
    <figcaption>Image asking to enable macros when opening report.xls</figcaption>
</figure>

On a real scenario, the contents of this image would serve the purpose of tricking the victim into enabling and executing the embedded macro containing malicious code.

I don't usually have to deal with macros on my reversing adventures, so I wanted to get the _full experience_ from this challenge. Thus, I will actually allow it to run the embedded macro to see what happens.

> WARNING: Never run an office macro outside a controled and isolated environment.

So, let's enable macros and allow to run. It will inmediately throw an error.

<figure class="align-center" style="width: 50%">
    <a href="/assets/images/posts/flareon7_ch4/error_macro_launch.png"><img src="/assets/images/posts/flareon7_ch4/error_macro_launch.png"></a>
    <figcaption>Error produced when attempting to launch macro</figcaption>
</figure>

From the title of this window, it appears that the embedded macro failing to run is coded in [Visual Basic for Applications (VBA)](https://en.wikipedia.org/wiki/Visual_Basic_for_Applications). Clicking in the `Ok` button will bring us into a fancy internal VBA IDE where we can explore all the components of the macro.

<figure class="align-center" style="width: 50%">
    <a href="/assets/images/posts/flareon7_ch4/report_VBAProject.png"><img src="/assets/images/posts/flareon7_ch4/report_VBAProject.png"></a>
    <figcaption>Project structure of report.xls VBA macro</figcaption>
</figure>

The object `ThisWorkbook` contains very little code that just jumps into the function `folderol` within `Sheet1`, which contains some amount of non-trivial code.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch4/original_vba.png"><img src="/assets/images/posts/flareon7_ch4/original_vba.png"></a>
    <figcaption>Sheet1 code initially found</figcaption>
</figure>

Also, we find the intersting form `F` with very suspicious data inside that appears to be used within `Sheet1`. We will export the `F` form for later usage (`Right click` -> `Export File...`).

<figure class="align-center" style="width: 50%">
    <a href="/assets/images/posts/flareon7_ch4/Forms_F.png"><img src="/assets/images/posts/flareon7_ch4/Forms_F.png"></a>
    <figcaption>Form F containing interesting data</figcaption>
</figure>

If we google for a couple minutes about VBA macros related to malware spread, we quickly find some _recent_ information about a technique known as VBA stomping. Essentially, this technique leverages a mismatch between the [p-code](https://en.wikipedia.org/wiki/Microsoft_P-Code) (the compiled [intermediate represenation](https://en.wikipedia.org/wiki/Intermediate_representation) bytecode of the VBA macro) and the code that is presented in clear text on the editor.

If you are interested, you can read more about VBA stomping in the following references:
- <https://github.com/outflanknl/EvilClippy>
- <https://medium.com/walmartglobaltech/vba-stomping-advanced-maldoc-techniques-612c484ab278>
- <https://vbastomp.com/>


This really appears to be what is going on with `report.xls`. Thus, we will continue by dumping the real p-code and obtaining its actual VBA representation so we can analyze/run it.

## Reconstruct document
I decided to (re)construct a valid document to embed and launch the fixed VBA macro within `Sheet1`. First of all, we create a new document, go to developer tab (will need to [enable developer tab on excel](https://support.microsoft.com/en-us/office/show-the-developer-tab-e1192344-5e56-4d45-931b-e5fd9bea2d45)) and click the first icon `Visual Basic`.

Now, we import the figure `F` saved before (Right click on project -> Import File... -> F.frm)
and copy-paste the contents of previous `ThisWorkBook` into the newly created one.

### Dump and decompile p-code
We dump the embedded p-code with [pcodedmp](https://github.com/bontchev/pcodedmp) into `macro.pcode` file.
```
pcodedmp -o macro.pcode report.xls
```

Among other info, the file `macro.pcode` contains the p-code formatted output for the code in `Sheet1`. Now we can _decompile_ the p-code back into VBA with [pcode2code](https://pypi.org/project/pcode2code/).
```
pcode2code -p macro.pcode > decompiled
```

If we take a look into the decompiled code we will see some small parts that have not been sucessfully decompiled. Despite that, we observe enough difference with the code presented to us initially, confirming our hypothesis of mismatching between the high level language representation and the embedded p-code. Essentially, `folderol` function now has a new buffer variable and some new code that plays with it.

There are also some minor errors in data types that we will need to fix if we want it to run. Moreover, we find a bunch of checks and protections that will have to be bypassed in order for the code to properly execute. Thus, let's copy the decompiled code into `Sheet1` in the newly created document and fix stuff as we go.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch4/raw_decompiled.png"><img src="/assets/images/posts/flareon7_ch4/raw_decompiled.png"></a>
    <figcaption>Sheet1 code raw decompiled from embedded p-code</figcaption>
</figure>


### Fix and clean decompiled code

From `rigmarole` function:
  - Remove `id_FFFE` parameter.

From `folderol` function:
  - Remove `id_FFFE` parameter.
  - In the usage of `wabbit` and `onzo` we see that they need to be array based types, so we just add a pair of missing parentheses to their type in its declaration.
  - `xertz` is no longer used, so we can remove its declaration and assignment.
  - The following body that failed to decompile:
```
Ld fn 
Sharp 
LitDefault 
Ld wabbit 
PutRec
```
essentially translates to `Put #fn, , wabbit`.

From `canoodle` function:
  - Remove `id_FFFE` parameter.
  - Return type has to be `byte()` instead of `Append`.
  - `kerfuffle` variable needs to be an array as well, so we add parentheses to its type.

> Note: Without knowing ~~nothing~~ too much about VBA or p-code, we can guess most fixes from the contents of the VBA code displayed in the original file, and just check that the corresponding p-code makes sense.

### Bypass checks and protections
- Remove everything before `rigmarole` function. We do not worry of internet checks and names, as we will bypass it.

- Entirely remove the condition using `GetInternetConnectedState` to check for internet connection.

- Then, we have a routine that appears to be doing some simple anti-vm checks. As nothing coming out from this part will be used, we can literally remove it entirely as well.

- Now we see `firkin` variable being loaded (with device name) and compared to the string returned by calling `rigmarole(onzo(3))`. We could be tempted to bypass the conditional, but we see that `firkin` is actually used afterwards, so we better get this value right by simply assigning to it the correct value `firkin = rigmarole(onzo(3))`


<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch4/bypass_protections.png"><img src="/assets/images/posts/flareon7_ch4/bypass_protections.png"></a>
    <figcaption>Bypass of checks and protections on decompiled folderol function</figcaption>
</figure>

> If you are curious, you could play with the integrated debugger to see what values are actually used and expected for the anti-vm checks and the `firkin` variable. Indeed, `firkin` will be loaded with the string `FLARE-ON`.

## Retrieve flag

After we have cleaned and fixed everything, as well as bypassing all the checks, we obtain the resulting code for `Sheet1`:

```
Function rigmarole(es As String) As String
  Dim furphy As String
  Dim c As Integer
  Dim s As String
  Dim cc As Integer
  furphy = ""
  For i = 1 To Len(es) Step 4
    c = CDec("&H" & Mid(es, i, 2))
    s = CDec("&H" & Mid(es, i + 2, 2))
    cc = c - s
    furphy = furphy + Chr(cc)
  Next i
  rigmarole = furphy
End Function
      
Function folderol()
  Dim wabbit() As Byte
  Dim fn As Integer: fn = FreeFile
  Dim onzo() As String
  Dim mf As String
  Dim buff(0 To 7) As Byte
  
  onzo = Split(F.L, ".")
  
  firkin = rigmarole(onzo(3))
  
  n = Len(firkin)
  For i = 1 To n
    buff(n - i) = Asc(Mid$(firkin, i, 1))
  Next
  
  wabbit = canoodle(F.T.Text, 2, 285729, buff)
  mf = Environ(rigmarole(onzo(0))) & rigmarole(onzo(11))
  Open mf For Binary Lock Read Write As #fn
    Put #fn, , wabbit
  Close #fn
  
  Set panuding = Sheet1.Shapes.AddPicture(mf, False, True, 12, 22, 600, 310)
End Function
      
Function canoodle(panjandrum As String, ardylo As Integer, s As Long, bibble As Variant) As Byte()
  Dim quean As Long
  Dim cattywampus As Long
  Dim kerfuffle() As Byte
  ReDim kerfuffle(s)
  quean = 0
  For cattywampus = 1 To Len(panjandrum) Step 4
    kerfuffle(quean) = CByte("&H" & Mid(panjandrum, cattywampus + ardylo, 2)) Xor bibble(quean Mod (UBound(bibble) + 1))
    quean = quean + 1
    If quean = UBound(kerfuffle) Then
      Exit For
    End If
  Next cattywampus
  canoodle = kerfuffle
End Function
```

Taking a quick look at the code, it is clear that the data present in the form `F` is being decrypted into a buffer that eventually gets written as an image into the spreadsheet itself.

One approach could have been to reimplement this decrypting algorithm externally, but as we already fixed the actual decompiled code and bypassed all checks, we will be able to simply run it and let it do all the work for us. Therefore, if we now just run the macro, we will get in the spreadsheet the resulting image with the flag:

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch4/flag.png"><img src="/assets/images/posts/flareon7_ch4/flag.png"></a>
    <figcaption>Decrypted image containing the flag</figcaption>
</figure>

Thus, we obtained the flag `thi5_cou1d_h4v3_b33n_b4d@flare-on.com` and completed the challenge.
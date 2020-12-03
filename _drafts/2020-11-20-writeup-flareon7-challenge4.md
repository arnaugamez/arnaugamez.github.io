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
  teaser: /assets/images/posts/flareon7_ch4/TODO.png
---

## Description
We are introduced to the challenge with the following message:

> Nobody likes analysing infected documents, but it pays the bills. Reverse this macro thrill-ride to discover how to get it to show you the key.

There is a single file `report.xls` to analyze, which is an excel document with the old windows format. From the challenge description and the kind of file we are given, we can be sure that we will have to deal with an excel macro.

## Analysis
If we open the document we find the following image within the spreadsheet.

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch4/report_initial_image.png"><img src="/assets/images/posts/flareon7_ch4/report_initial_image.png"></a>
    <figcaption>AAAA</figcaption>
</figure>

On a real scenario, the contents of this image would serve the purpose of tricking the victim into executing the embedded macro containing malicious code.

I don't usually have to deal with macros on my reversing adventures, so I wanted to get the _full experience_ from this challenge. Thus, I will actually allow it to run the embedded macro to see what happens.

> WARNING: Never run an office macro outside a controled and isolated environment.

So, let's enable macros and allow to run. It will inmediately throw an error.

<figure class="align-center" style="width: 50%">
    <a href="/assets/images/posts/flareon7_ch4/error_macro_launch.png"><img src="/assets/images/posts/flareon7_ch4/error_macro_launch.png"></a>
    <figcaption>AAAA</figcaption>
</figure>

From the title of this window, it appears that the embedded macro which is failing to run is coded in [Visual Basic for Applications (VBA)](https://en.wikipedia.org/wiki/Visual_Basic_for_Applications). Clicking in the `Ok` button will bring us into a fancy internal VBA IDE where we can explore all the components of the macro.

<figure class="align-center" style="width: 50%">
    <a href="/assets/images/posts/flareon7_ch4/report_VBAProject.png"><img src="/assets/images/posts/flareon7_ch4/report_VBAProject.png"></a>
    <figcaption>AAAA</figcaption>
</figure>

There is some code in `Sheet1` which appears to not work correctly. Also, we find the intersting form `F` with very suspicious data inside. We will export the `F` form for later usage (`Right click` -> `Export File...`).

<figure class="align-center" style="width: 50%">
    <a href="/assets/images/posts/flareon7_ch4/Forms_F.png"><img src="/assets/images/posts/flareon7_ch4/Forms_F.png"></a>
    <figcaption>AAAA</figcaption>
</figure>

If we google for a couple minutes about VBA macros related to malware spread, we quickly find some _recent_ information about a technique known as VBA stomping. Essentially, this technique leverages a mismatch between the [p-code](https://en.wikipedia.org/wiki/Microsoft_P-Code) (the compiled [intermediate represenation](https://en.wikipedia.org/wiki/Intermediate_representation) bytecode of the VBA macro) and the code that is presented in clear text on the editor.

If you are interested, you can read more about VBA stomping in the following references:
- <https://github.com/outflanknl/EvilClippy>
- <https://medium.com/walmartglobaltech/vba-stomping-advanced-maldoc-techniques-612c484ab278>
- <https://vbastomp.com/>


This really appears to be what is going on with `report.xls`. Thus, we will continue by dumping the real p-code and obtaining its actual VBA representation so we can analyze/run it.

## Reconstruct document
I decided to (re)construct a valid document to embed and launch the fixed VBA macro.

Create a new document. Will need to enable developer tab on excel: https://support.microsoft.com/en-us/office/show-the-developer-tab-e1192344-5e56-4d45-931b-e5fd9bea2d45

Go to developer tab and click the first icon on VBA.

Import the figure (Right click on project. Import File... -> F.frm)
Copy-paste contents from/to ThisWorkBook:

### Dump and decompile pcode
Dump pcode with [pcodedmp](https://github.com/bontchev/pcodedmp) into `macro.pcode` file.
```
pcodedmp -o macro.pcode report.xls
```

Among other info, the file `macro.pcode` contains the following pcode formatted output for the embedded macro:

```
snippet of macro.pcode
```

Now we can _decompile_ the pcode back into VBA with [pcode2code](https://pypi.org/project/pcode2code/).
```
pcode2code -p macro.pcode > decompiled
```

If we take a look into the decompiled code we will see some bits that have not been sucessfully decompiled. Despite that, there is some significant difference with the code presented to us initially, so it confirms our hypothesis of mismatching between high level language representation and the embeded maco pcode. Namely `folderol` function now has a buffer variable and some new code that plays with it.

Also there are some minor errors in data types and so that we will need to fix if we want it to run. So, let's copy the decompiled code into Sheet1 (the corresponding) in the newly created document and we will fix in there as we go.

### Fix and clean decompiled code

- Remove `id_FFFE` parameter from rigmarole.

- Remove `id_FFFE` parameter from folderol.
-On folderol, in the usage of `wabbit` and `onzo` we see that they need to be array based types, so we just add a pair of missing parentheses to them on its declaration.
- xertz is no longer used, so we can remove its declaration and assignment.
- The following body that failed to decompile is the corresponding to Line #68 in the pcode above.
```
Ld fn 
Sharp 
LitDefault 
Ld wabbit 
PutRec
```
which essentially translates to `Put #fn, , wabbit`.

- Remove `id_FFFE` parameter from canoodle.
- Now `canoodle` function needs to return `byte()` instead of `Append`.
- `kerfuffle` variable needs to be array as well so we add parentheses.

> Note: Without knowing too much about VBA or pcode, we can guess most fixes from the contents of the vba code displayed in the original file, and just check that the corresponding pcode makes sense.

### Bypass checks and protections
- Remove everything before `rigmarole` function. We do not worry of internet checks and names, as we will bypass it.



- Entire remove the `If` that sets and checks `GetInternetConnectedState` and ends if not connection.

- Then, it seems a vm check routine. As nothing that comes from it is used, we can literally remove it as well.

- Now we see `firkin` variable created being compared into the string resulting from the call `rigmarole(onzo(3))`. We could be tempted to bypass the conditional that leads to close again, but we see that `firkin` is used before, so we better get this value right by simply assigning to it the correct value `firkin = rigmarole(onzo(3))`


<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch4/bypass_protections.png"><img src="/assets/images/posts/flareon7_ch4/bypass_protections.png"></a>
    <figcaption>AAAA</figcaption>
</figure>

MAYBE SAY SOMETHING ABOUT THE VM CHECKS WHAT IT CHECKS AGAINST.
ALSO SOMETHING ABOUT THE RIGHT VALUE OF THE FIRKIN

## Retrieve flag

Now the code is fixed and we bypassed all checks.

The resulting code is here:
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

If we now run the macro, we will get in the spreadsheet the resulting image with the flag:

<figure class="align-center">
    <a href="/assets/images/posts/flareon7_ch4/flag.png"><img src="/assets/images/posts/flareon7_ch4/flag.png"></a>
    <figcaption>AAAA</figcaption>
</figure>

Thus, we obtained the flag `thi5_cou1d_h4v3_b33n_b4d@flare-on.com` and completed the challenge.
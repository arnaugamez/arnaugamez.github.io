---
title: "Pwning F@st 5655v2AC router"
categories:
  - Blog
tags:
  - router
  - ARP
  - WoL
  - Wake on Lan
---

**Long story short**: I set up a computer lab at home and wanted to enable [Wake-on-Lan](https://en.wikipedia.org/wiki/Wake-on-LAN) so I could power on the machine remotely whenever I needed it. Looked like easy task, but ended up having to hack into my home router in order to add a static entry on the [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) table for the machine, since the ethernet connection stops delivering power after the entry for this computer gets flushed from the ARP table cache (I assume due to built-in power saving management) and the included router control panel did not have that option, even if accessing as admin user.










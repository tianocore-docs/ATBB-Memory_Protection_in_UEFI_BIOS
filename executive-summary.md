# Executive Summary
## Introduction
**Data execution protection (DEP)** is intended to prevent an application or service from executing code from a non-executable memory region. This helps prevent certain exploits that store code via a buffer overflow. [WindowsHeap][1] shows 4 of 7 exploitation techniques that can be mitigated by DEP and ASLR (Address Space Layout Randomization). [DEP] also shows 14 of 19 exploits from popular exploit kits that fail with DEP enabled. Besides Windows, the Unix/Linux community also has similar non-executable protection [PaX].

In [MemMap], we discussed DEP and the limitation of enabling DEP in UEFI firmware. In [SecurityEnhancement], we only discussed the DEP for protecting the stack and setting the not-present page for detecting NULL address accesses and as the guard page. In this document we will have a more comprehensive dicsussion of the DEP adoption in the current UEFI firmware to harden the pre-boot phase.

[WindowsHeap]: Preventing the exploitation of user mode heap corruption vulnerabilities, 2009, 

[1]: https://blogs.technet.microsoft.com/srd/2009/08/04/preventing-the-exploitation-of-user-mode-heap-corruption-vulnerabilities/ "WindowsHeap"


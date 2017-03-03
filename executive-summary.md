# Executive Summary
## Introduction
**Data execution protection (DEP)** is intended to prevent an application or service from executing code from a non-executable memory region. This helps prevent certain exploits that store code via a buffer overflow. [[WindowsHeap][1]] shows 4 of 7 exploitation techniques that can be mitigated by DEP and ASLR (Address Space Layout Randomization). [[DEP][2]] also shows 14 of 19 exploits from popular exploit kits that fail with DEP enabled. Besides Windows, the Unix/Linux community also has similar non-executable protection [[PaX][3]].

In [[MemMap][4]], we discussed DEP and the limitation of enabling DEP in UEFI firmware. In [[SecurityEnhancement][5]], we only discussed the DEP for protecting the stack and setting the not-present page for detecting NULL address accesses and as the guard page. In this document we will have a more comprehensive dicsussion of the DEP adoption in the current UEFI firmware to harden the pre-boot phase.



[1]: https://blogs.technet.microsoft.com/srd/2009/08/04/preventing-the-exploitation-of-user-mode-heap-corruption-vulnerabilities/ "WindowsHeap"

[2]: http://media.blackhat.com/bh-us-12/Briefings/M_Miller/BH_US_12_Miller_Exploit_Mitigation_Slides.pdf "DEP"

[3]: https://pax.grsecurity.net/ "Pax"

[4]: https://github.com/tianocore-docs/Docs/raw/master/White_Papers/A_Tour_Beyond_BIOS_Memory_Map_And_Practices_in_UEFI_BIOS_V2.pdf "MemMap"

[5]: https://github.com/tianocore-docs/Docs/raw/master/White_Papers/A_Tour_Beyond_BIOS_Securiy_Enhancement_to_Mitigate_Buffer_Overflow_in_UEFI.pdf "SecurityEnhancement"


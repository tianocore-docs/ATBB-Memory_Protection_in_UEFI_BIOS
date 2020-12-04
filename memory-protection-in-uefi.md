<!--- @file
  Memory Protection in UEFI

  Copyright (c) 2008-2017, Intel Corporation. All rights reserved.<BR>

  Redistribution and use in source (original document form) and 'compiled'
  forms (converted to PDF, epub, HTML and other formats) with or without
  modification, are permitted provided that the following conditions are met:

  1) Redistributions of source code (original document form) must retain the
     above copyright notice, this list of conditions and the following
     disclaimer as the first lines of this file unmodified.

  2) Redistributions in compiled form (transformed to other DTDs, converted to
     PDF, epub, HTML and other formats) must reproduce the above copyright
     notice, this list of conditions and the following disclaimer in the
     documentation and/or other materials provided with the distribution.

  THIS DOCUMENTATION IS PROVIDED BY TIANOCORE PROJECT "AS IS" AND ANY EXPRESS OR
  IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
  MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO
  EVENT SHALL TIANOCORE PROJECT  BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
  SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
  PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS;
  OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
  WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR
  OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS DOCUMENTATION, EVEN IF
  ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

-->

# Memory Protection in UEFI
In the white paper [[MemMap][1]], we discussed to how to report the runtime memory attribute by using `EFI_MEMORY_ATTRIBUTES_TABLE`, so that OS can apply the protection for the runtime code and data. This may bring some compatibility concerns if we choose to adopt the full DEP protection for the entire UEFI memory.


In order to resolve the compatibility concerns, we can define a policy-based setting to enable partial NX and RO protection for the UEFI memory region. The detailed information will be discussed below.
![](/media/Fig4 - UEFI memory protection.jpg)

###### Figure 4 - UEFI memory protection

## Protection for PE image
The DXE core may apply a pre-defined policy to set up the NX attribute for the PE data region and the RO attribute for the PE code region.

1. The image is loaded by the UEFI boot service - `LoadImage()`. If an image is loaded in some other way, the DXE core does not have such knowledge and the DXE core cannot apply any protection.
2. The image section is page aligned. If an image is not page aligned, the DXE core cannot apply the page level protection.
3. The protection policy can be based upon a PCD `PcdImageProtectionPolicy`. (https://github.com/tianocore/edk2/blob/master/MdeModulePkg/MdeModulePkg.dec) Whenever a new image is loaded, the DxeCore checks the source of the image and then decides the policy of the protection. The policy could be to enable the protection if the sections are aligned, or disable the protection. The platform may choose the policy based upon the need. For example, if a platform thinks the image from the firmware volume should be capable of being protection, it can set protection for IMAGE_FROM_FV. But if a platform is not sure about a PCI option ROM or a file system on disk, it can set no-protection.

There are assumptions for the PE image protection in UEFI:

1. [Same as SMM] The PE code section and data sections are not merged. If those 2 sections are merged, a #PF exception might be generated because the CPU may try to write a RO data in data section or execute a NX instruction in the code section.
2. [Same as SMM] The PE image can be protected if it is page aligned. There should not be any self-modifying-code in the code region. If there is, a platform should not set this PE image to be page aligned.
3. A platform may not disable the XD in the DXE phase. If a platform disables the XD in the DXE phase, the X86 page table will become invalid because the XD bit in page table becomes a RESERVED bit. The consequence is that a #PF exception will be generated. If a platform wants to disable the XD bit, it must happen in the PEI phase.

In EDK II, the DXE core image services calls `ProtectUefiImage()` on image load and `UnprotectUefiImage()` on image unload. (https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/Dxe/Image/Image.c) Then `ProtectUefiImageCommon()` (https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/Dxe/Misc/MemoryProtection.c) calls `GetUefiImageProtectionPolicy()` to check the image source and protection policy and parses PE alignment. If all checks pass, `SetUefiImageProtectionAttributes()` calls `SetUefiImageMemoryAttributes()`. Finally, `gCpu->SetMemoryAttribute()` sets **EFI_MEMORY_XP** or **EFI_MEMORY_RO** for the new loaded image , or clears the protection for the old unloaded image. When the CPU driver gets the memory attribute setting request, it updates page table.

The X86 CPU driver https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/CpuDxe/CpuDxe.c `CpuSetMemoryAttributes ()` calls  https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/CpuDxe/CpuPageTable.c, `AssignMemoryPageAttributes()` to setup page table.

The ARM CPU driver https://github.com/tianocore/edk2/blob/master/ArmPkg/Drivers/CpuDxe/CpuMmuCommon.c `CpuSetMemoryAttributes()` also has similar capability.

If an image is loaded before CPU_ARCH protocol is ready, the DXE core just skips the setting. Later these images protection will be set in CPU_ARCH callback function - `MemoryProtectionCpuArchProtocolNotify()`(https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/Dxe/Misc/MemoryProtection.c).

In `ExitBootServices` event, `MemoryProtectionExitBootServicesCallback() `(https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/Dxe/Misc/MemoryProtection.c) is invoked to unprotect the runtime image, because the runtime image code relocation need write code segment at `SetVirtualAddressMap()`.

## Protection for stack and heap
[[UEFI][2]] specification allows
>"Stack may be marked as non-executable in identity mapped page tables."

As such, we set up the NX stack (https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/DxeIplPeim/X64/VirtualMemory.c, `CreateIdentityMappingPageTables()`).



The heap protection is based upon the policy, because we already observed some unexpected usage in [[MemMap][1]] white paper. A platform needs to  configure a PCD `PcdDxeNxMemoryProtectionPolicy`
(https://github.com/tianocore/edk2/blob/master/MdeModulePkg/MdeModulePkg.dec) to indicate which type of memory can be set to NX in the page table. The DxeCore `ApplyMemoryProtectionPolicy()` (https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/Dxe/Misc/MemoryProtection.c) consumes the PCD after the memory allocation service and sets NX attribute for the allocated memory by using CPU_ARCH protocol.

Before CPU_ARCH protocol is ready, the protection takes no effect. In CPU_ARCH callback function - `MemoryProtectionCpuArchProtocolNotify() `(https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/Dxe/Misc/MemoryProtection.c), the `InitializeDxeNxMemoryProtectionPolicy()` is called to get current memory map and setup the NX protection.


In addition, we may use some special techniques, such as the guard page, to apply the protection for the allocated memory in order to detect a buffer overflow. This is discussed in [[SecurityEnhancement][3]] white paper.

## Life cycle of the protection
The UEFI image protection starts when the CpuArch protocol is ready. The UEFI runtime image protection is torn down at `ExitBootServices()`, the runtime image code relocation need write code segment at `SetVirtualAddressMap()`. We cannot assume OS/Loader has taken over page table at that time.

The UEFI heap protection also starts when the `CpuArch` protocol is ready.

The UEFI stack protection starts in `DxeIpl`, because the region is fixed and it can set directly.


The UEFI firmware does not own page tables after `ExitBootServices()`, so the OS would have to relax protection of runtime code pages across `SetVirtualAddressMap()`, or delay setting protections on runtime code pages until after `SetVirtualAddressMap()`. OS may set protection on runtime memory based upon EFI_MEMORY_ATTRIBUTES_TABLE later.

## Size Overhead

1. Runtime memory overhead (visible to OS)
: The size overhead of the runtime PE image is the same as the overhead of the SMM PE image.  If a platform has n runtime images, the average amount overhead is `6K * n`.
2. Boot time memory overhead (invisible to OS)
: The size of the overhead for the boot time PE image is the same as the overhead of the SMM PE image. If a platform has n boot time images, the average overhead is `6K * n`.

If the NX protection for data is enabled, the size of the page table is increased because we need set fine granularity page level protection.

The size overhead of the boot time page table is also same as for the SMM static page table. Please refer to the SMM section for the size calculation based upon the 1G paging capability and max supported address bit.

## Limitation
The protection in the UEFI is limited to the PE image and the stack at this moment because of the compatibility concerns. The limitations of the UEFI memory protection are:

1. Not all images are protected to be NX and RO. The protection is based upon the policy.
2. Not all heap regions are protected to be NX due to the compatibility concern. We observed that both Windows boot loader and Linux boot loader may use the LoaderData type for the code. The heap protection is based upon the policy.
3. [Same as SMM] The protection cannot resist ROP attack.
4. [Same as SMM] Not all important data structures are set to ReadOnly.

## Compatibility Consideration
A platform may need to evaluate and select the image protection policy based upon the capability of the platform image, Option ROM, and OS loader. For platform images, the Compatibility Support Module (CSM) and the EDK-I Compatibility Package (ECP) modules should be considered. If a platform observes the compatibility issues, it should choose 1) to disable the protection, or 2) to fix the compatibility issue and enable the protection.

## Call for action
In order to support UEFI memory protection, the firmware need configure UEFI driver to be page aligned:

1. Override link flags below to support UEFI runtime attribute table, so that OS can protect the runtime memory.
```
[BuildOptions.IA32.EDKII.DXE_RUNTIME_DRIVER, BuildOptions.X64.EDKII.DXE_RUNTIME_DRIVER]
MSFT:*_*_*_DLINK_FLAGS = /ALIGN:4096
GCC:*_*_*_DLINK_FLAGS = -z common-page-size=0x1000
```
2. Override link flags below to support UEFI memory protection.
```
[BuildOptions.common.EDKII.DXE_DRIVER,
BuildOptions.common.EDKII.DXE_CORE,
BuildOptions.common.EDKII.UEFI_DRIVER, BuildOptions.common.EDKII.UEFI_APPLICATION]
MSFT:*_*_*_DLINK_FLAGS = /ALIGN:4096
GCC:*_*_*_DLINK_FLAGS = -z common-page-size=0x1000
```
3.  Evaluate if the UEFI memory size is big enough to hold the split page table.

4. Evaluate if the DXE image can be protected.

5. Set proper `gEfiMdeModulePkgTokenSpaceGuid.PcdImageProtectionPolicy`.

6. Set proper `gEfiMdeModulePkgTokenSpaceGuid.PcdDxeNxMemoryProtectionPolicy`.

#### Summary
This section introduces the memory protection in UEFI.

[1]: https://github.com/tianocore-docs/Docs/raw/master/White_Papers/A_Tour_Beyond_BIOS_Memory_Map_And_Practices_in_UEFI_BIOS_V2.pdf "MemMap"
[2]: http://uefi.org "UEFI"
[3]: https://github.com/tianocore-docs/Docs/raw/master/White_Papers/A_Tour_Beyond_BIOS_Securiy_Enhancement_to_Mitigate_Buffer_Overflow_in_UEFI.pdf "Security Enhancment"
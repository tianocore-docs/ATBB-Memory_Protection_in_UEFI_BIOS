<!--- @file
  Memory Protection in SMM

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

# Memory Protection in SMM

The SMM is an isolated execution environment according to Intel(R) 64 and IA-32 Architectures Software Developer's Manual [[IA32SDM][1]]. The UEFI Platform Initialization [[PI][2]] specification volume 4 defines the SMM infrastructure. Figure 1 shows the SMM memory protection. **RO** designates read-only memory. **XD **designates execution-disabled memory.

![](/media/Fig1- SMRAM memory protection.jpg)

###### Figure 1 - SMRAM memory protection

## Protection for PE image
In UEFI/PI firmware, the SMM image is a normal PE/COFF image loaded by the SmmCore. If a given section of the SMM image is page aligned, it may be protected according to the section attributes, such as read-only for the code and non-executable for data. See the top right of Figure 1.

In EDK II, the PiSmmCore (https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/PiSmmCore/MemoryAttributesTable.c) checks the PE image alignment and builds an `EDKII_PI_SMM_MEMORY_ATTRIBUTES_TABLE ` (https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Include/Guid/PiSmmMemoryAttributesTable.h) to record such information. If the PI SMM image is not page aligned, this table will not be published. If the `EDKII_PI_SMM_MEMORY_ATTRIBUTES_TABLE` is published, that means the `EfiRuntimeServicesCode` contains only code and it is ``EFI_MEMORY_RO``, and the `EfiRuntimeServicesData` contains only data and it is `EFI_MEMORY_XP`.

Later the PiSmmCpu driver (https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/SmmCpuMemoryManagement.c)` SetMemMapAttributes()` API consumes the ``EDKII_PI_SMM_MEMORY_ATTRIBUTES_TABLE`` and sets the page table attribute.

There are several assumptions to support the PE image protection in SMM:

1. The PE code section and data sections are not merged. If those 2 sections are merged, a #PF exception might be generated because the CPU might try to write a RO data item in the data section or execute a non-executable (NX) instruction in code section.
2. The PE image can be protected if it is page aligned. There should not be any self-modified-code in the code region. If there is, a platform should not set this PE image to be page aligned.

A platform may disable the XD in the UEFI environment, but this does not impact the SMM environment. The SMM environment may choose to always enable the XD upon SMM entry, and restore the XD state at the SMM exit point.

## Protection for stack and heap
The PiSmmCore maintains a memory map internally. (https://github.com/tianocore/edk2/blob/master/MdeModulePkg/Core/PiSmmCore/Page.c) If an SMM module allocates the data with `EfiRuntimeServicesCode`, this data is marked as the code page. If the SMM module allocates the data with `EfiRuntimeServicesData`, this data is marked as the data page. This information is also exposed via the `EDKII_PI_SMM_MEMORY_ATTRIBUTES_TABLE`.

The same RO and XD policy is also applied to the normal SMM data region, such as stack and heap.

## Protection for critical CPU status
Besides the PE image, the Intel X86 architecture has some special architecture-specific regions that need to be protected as well.

### SMM EntryPoint
When a hardware SMI occurs, the Intel X86 CPU jumps to an SMM entry point in order to execute the code at this location. This SMM entry point is not inside of a normal PE image, so we also need to protect this region. See the bottom right of figure 2.

According to [[IA32SDM][1]], the SMM entry point is at a fixed offset from SMBASE. In EDK II, the SMBASE and the SMM save state area are allocated at https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/PiSmmCpuDxeSmm.c `PiCpuSmmEntry()`. Both the SMM entry point and the SMM save state are allocated as a CODE page. Later https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/SmmCpuMemoryManagement.c  `PatchSmmSaveStateMap()` patches the SMM entry point to be read-only and the SMM save state to be non-executable.

### GDT/IDT
The GDT defines the base address and the limit of a code segment or a data segment. If the GDT is updated, the code might be redirected to a malicious region. As such, the GDT should be set to read-only.

The IDT defines the entry point of the exception handler. If the IDT is updated, the malicious code may trigger an exception and jump to a malicious region. As such, the IDT should be set to read-only as well.

This work is done by `PatchGdtIdtMap()` at https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/X64/SmmFuncsArch.c.

However, the IA32 version GDT cannot be set to read-only if the stack guard feature is enabled. (https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/Ia32/SmmFuncsArch.c) The reason is that the IA32 stack guard needs to use a "_task switch_" to switch the stack, and the task switch needs to write the GDT and Task-State Segment (TSS). The X64 version of the GDT does not have such a problem because the X64 stack guard uses "_interrupt stack table (IST)_" to switch the stack. For details of the stack switch and exceptions, please refer to [[IA32SDM][1]].

### Page Table
In an X86 CPU, we rely on the page table to set up the read-only or non-executable region. In order to prevent the page table itself from being updated, we may need to set the page table itself to be read-only.

The work is done at https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/X64/PageTbl.c `SetPageTableAttributes()`.

However, setting a page table to be read-only may break the original dynamic paging feature in SMM. There is a (PCD) `PcdCpuSmmStaticPageTable` to determine if the platform wants to enable the static page table or the dynamic page table.

If `PcdCpuSmmStaticPageTable` is FALSE, the PiSmmCpu uses the original dynamic paging policy, namely the the PiSmmCpu only sets 4GiB paging by default. If the PiSmmCpu needs to access above 4GiB memory locations, a page fault exception (#PF) exception is triggered and an above-4GiB mapping is created in the page fault handler.

If `PcdCpuSmmStaticPageTable` is TRUE, the PiSmmCpu will try to set the read-only attribute for the page table.

Figure 2 shows the mapping of the protection.

![](/media/Fig2 - Mapping of Protection in SMM.jpg)

###### Figure 2 - Mapping of Protection in SMM

## Life cycle of the protection
In a normal boot, the page table based protection is configured by the PiSmmCpu driver just after the SmmReadyToLock event by `PerformRemainingTasks()` at https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/PiSmmCpuDxeSmm.c. All read-only data must be ready before `SmmReadyToLock`.

In an S3 resume, the protection is disabled during SMBASE relocation because the PiSmmCpu needs to set up the environment. The PiSmmCpu uses SmmS3Cr3, which is generated by `InitSmmS3Cr3()` at https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/X64/SmmProfileArch.c  with 4G paging only. After the SMBASE relocation is done, all the protection takes effect up receipt of the next SMI by `PerformPreTasks()` at https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/PiSmmCpuDxeSmm.c.

If there is an additional lock that needs to be set, it can be done in `SmmCpuFeaturesCompleteSmmReadyToLock()` API (defined in https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/Include/Library/SmmCpuFeaturesLib.h).

## SMRAM Size Overhead
### PE image
In order to protect the PE code and data sections, we must set the PE image section alignment to be 4K.

In EDK II, the default PE image alignment is 0x20 bytes. Assuming one PE image has 3 sections (1 header, 1 code section, 1 data section), average overhead for one PE image is `(4K * 3) / 2 = 6K`.

If a platform has n SMM images, the average of the overhead is `6K * n`.

### Page Table
In order to protect the page table itself, we must use the static page table instead of the dynamic on-demand page table.

The size of the dynamic paging is fixed. We need 6 fixed pages (24K) and 8 on-demand pages (32K). The total size of the page table is 56K in this case.

The size of the static page table depends upon 2 things: 1) 1G paging capability, 2) max supported address bit. A rough estimation is below:
1. If 1G paging is supported,
* 32 bit addressing need (1+1+4) pages = 24K. (still use 2M paging for below 4G memory)
* 39 bit addressing need (1+1+4) pages = 24K.
* 48 bit addressing need (1+512) pages = 2M.
* If 1G paging is not supported, 2M paging is used.
* 32 bit addressing need (1+1+4) pages = 24K.
* 39 bit addressing need (1+1+512) pages = 2M.
* 48 bit addressing need (1+512+512*512) pages = 1G. < - This seems ****not**** acceptable.


The maximum address bit is determined by the (CPU_HOB) if it is present, or the physical address bit returned by the CPUID instruction if the CPU_HOB is not present. (https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/X64/PageTbl.c, `CalculateMaximumSupportAddress()`) A platform may set the CPU_HOB based upon the addressing capability of the memory controller or the CPU.

## Performance Overhead
1. The SMRAM protection setup is a one-time activity. It happens just after the SmmReadyToLock event. We do not observe too much impact to the system firmware boot performance. The activity only takes some small number of milliseconds.

2. The SMRAM runtime protection is based upon the page table. No additional CPU instruction is needed. As such, there is zero SMM runtime performance impact to have this protection.

## Non SMRAM access in SMM
Besides the SMRAM, the SMM memory protection also limits the access to the non-SMRAM region.

First, the non-SMRAM region must be set to be non-executable because the SMM entities should not call any code outside SMRAM. Code outside of SMRAM might be controlled by malicious software.

This protection work is done by `InitPaging()` at https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/SmmProfile.c


Second, because of the security concerns regarding SMM entities accessing VMM memory, [[WindowsWSMT][3]] [[Wsmt.docx][4]] and [[MicrosoftHV][5]] introduced the Windows SMM Security Mitigations Table (WSMT). A platform needs to report the WSMT table in order to declare that the SMI handler will validate the SMM communication buffer.

As we discussed in [[SecureSmmComm][6]], the SMI handler should check if the SMM communication buffer is from a fixed region, (EfiReservedMemoryType/ EfiACPIMemoryNVS/ EfiRuntimeServicesData/ EfiRuntimeServicesCode). However, this is a passive check. If a SMI handler does not include such a check, it is hard to detect.

A better way is to use an active check. The PiSmmCpu driver sets the non-fixed DRAM region (EfiLoaderCode/ EfiLoaderData/ EfiBootServicesCode/ EfiBootServicesData/ EfiConventionalMemory/ EfiUnusableMemory/ EfiACPIReclaimMemory) to be not-present in the page tables after the SmmReadyToLock event.

As such, if a platform SMI handler does not include the check recommended in [[SecureSmmComm][6]], the system will get #PF exception within SMM on such an attack.

This protection work is done by `SetUefiMemMapAttributes()` at https://github.com/tianocore/edk2/blob/master/UefiCpuPkg/PiSmmCpuDxeSmm/SmmCpuMemoryManagement.c.

Figure 3 shows final image layout.

 ![](/media/Fig3 - Page table enforced memory layout.jpg)

###### Figure 3 - Page table enforced memory layout

The assumption for non-SMRAM access in SMM is described in [[SecureSmmComm][6]].
Besides that, this solution assumes that all DRAM regions are added to the Global Coherency Domain (GCD) management before EndOfDxe, so that the UEFI memory map can return all DRAM regions. If there are more regions added to the GCD after EndOfDxe, those regions are not set to not-present in the page table.
NOTE: The SMM does not set the not-present bit for the GCD **EfiGcdMemoryTypeNonExistent** memory, because this type of memory may be converted to the other types, such as **EfiGcdMemoryTypeReserved**, or **EfiGcdMemoryTypeMemoryMappedIo**, which might be accessed by the SMM later.

## Limitation
Setting up RO and NX attribute for SMRAM is a good enhancement to prevent a code overriding attack. However it has some limitations:

1. It cannot resist a Return-Oriented-Programming (ROP) attack. [[ROP][8]]. We might need ASLR to mitigate the ROP attack. [[ASLR][7]] With the code region randomized, an attacker cannot accurately predict the location of instructions in order to leverage gadgets.
2. Not all important data structure are set to Read-Only. This is the current SMM driver limitation. The SMM driver can be updated to allocate the important structures to be read-only instead of a read-write global variable.

To set not-present bit for non-fixed DRAM region in SmmReadyToLock is a good enhancement to enforce the protection policy. However, it cannot cover below cases:

1. Memory Hot Plug. Take a server platform as the example, A RAS server may hot plug more DRAM during OS runtime, and rely on SMM to initialize those DRAM. This SMM Memory Initialization module may need access the DRAM for the memory test.
2. Memory Mapped IO (MMIO). Ideally, not all MMIO regions are configured to be accessible to SMM. Some MMIO BARs are important such as VTd or SPI controller.  VTd BAR is important because OS need setup VTd to configuration the DMA protection. SPI controller BAR is important because BIOS SMM handler need access it to program the flash device. It should be a platform policy to configure which one should be accessible. The SMI handler must consider the case that the MMIO BAR might be modified by the malicious software and check if the MMIO BAR is in the valid region.

## Compatibility Considerations
1. So far, we have not observed self-modified-code in SMM image or executable code in data section. As such, we believe the PE image protection is compatible.

2. The protection for the SMM communication buffer may cause a #PF exception in SMM if the SMI handler does not perform the check recommended in [[SecureSmmComm][6]].

3.  Some legacy Compatibility Support Module (CSM) drivers may need co-work with SMM module. Then the SMM driver need access the legacy region. As such these memory regions should be allocated as ReservedMemory, such as BIOS data area (BDA) or extended BIOS data area (EBDA).

## Call for action
In order to support SMM memory protection, the firmware need configure SMM driver to be page aligned:
1. Override link flags below to support SMM memory protection.
  ```
        [BuildOptions.common.EDKII.DXE_SMM_DRIVER,
        BuildOptions.common.EDKII.SMM_CORE]
        MSFT:*_*_*_DLINK_FLAGS = /ALIGN:4096
      GCC:*_*_*_DLINK_FLAGS = -z common-page-size=0x1000
  ```

2. Evaluate if SMRAM size is big enough.

#### Summary
This section introduces the memory protection in SMM.

[1]: https://software.intel.com/en-us/articles/intel-sdm "IA32SDM"
[2]: http://uefi.org "PI Spec"
[3]: https://msdn.microsoft.com/en-us/library/windows/hardware/dn495660(v=vs.85).aspx#wsmt "WindowsWSMT"
[4]: http://download.microsoft.com/download/1/8/A/18A21244-EB67-4538-BAA2-1A54E0E490B6/WSMT.docx "WindowsWSMT docx"
[5]: https://msdn.microsoft.com/en-us/library/windows/hardware/dn614617 "MicrosoftHV"
[6]: https://github.com/tianocore-docs/Docs/raw/master/White_Papers/A_Tour_Beyond_BIOS_Secure_SMM_Communication.pdf "SecureSmmComm"
[7]: https://en.wikipedia.org/wiki/Address_space_layout_randomization "ASLR"
[8]: https://en.wikipedia.org/wiki/Return-oriented_programming "ROP"

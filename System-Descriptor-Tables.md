<!--
author: zhuoliang
head: http://pingodata.qiniudn.com/jockchou-avatar.jpg
date: 2016-07-26
title: Descriptor Tables
tags: System
category: Resource
status: publish
-->

## Introduction ##

The x86 architecture has a number of different descriptor tables that are used by the processor to handle things like memory management (GDT), interrupt dispatching (IDT), and so on. In addition to processor-level descriptor tables, the Windows operating system itself also includes a number of distinct software-level descriptor tables, such as the SSDT. The majority of these descriptor tables are heavily relied upon by the operating system and therefore represent a tantalizing target for use in backdoors. Like the function hooking technique described in 2.1.1, all of the techniques presented in this subsection have been known about for a significant amount of time. The authors have attempted, when possible, to identify the origins of each technique. 

## IDT ##

 The Interrupt Descriptor Table (IDT) is a processor-relative structure that is used when dispatching interrupts. Interrupts are used by the processor as a means of interrupting program execution in order to handle an event. Interrupts can occur as a result of a signal from hardware or as a result of software asserting an interrupt through the int instruction[23]. The IDT contains 256 descriptors that are associated with the 256 interrupt vectors supported by the processor. Each IDT descriptor can be one of three types of gate descriptors (task, interrupt, trap) which are used to describe where and how control should be transferred when an interrupt for a particular vector occurs. The base address and limit of the IDT are stored in the idtr register which is populated through the lidt instruction. The current base address and limit of the idtr can be read using the sidt instruction.

The concept of an IDT hook has most likely been around since the origin of the concept of interrupt handling. In most cases, an IDT hook works by redirecting the procedure entry point for a given IDT descriptor to an alternative location. Conceptually, this is the same process involved in hooking any function pointer (which is described in more detail in 2.5). The difference comes as a result of the specific code necessary to hook an IDT descriptor.

On the x86 processor, each IDT descriptor is an eight byte data structure. IDT descriptors that are either an interrupt gate or trap gate descriptor contain the procedure entry point and code segment selector to be used when the descriptor's associated interrupt vector is asserted. In addition to containing control transfer information, each IDT descriptor also contains additional flags that further control what actions are taken. The Windows kernel describes IDT descriptors using the following structure: 

  	kd> dt _KIDTENTRY
     	+0x000 Offset           : Uint2B
  	    +0x002 Selector         : Uint2B
 	    +0x004 Access           : Uint2B
 	    +0x006 ExtendedOffset   : Uint2B

In the above data structure, the `Offset` field holds the low 16 bits of the procedure entry point and the ExtendedOffset field holds the high 16 bits. Using this knowledge, an IDT descriptor could be hooked by redirecting the procedure entry point to an alternate function. The following code illustrates how this can be accomplished:

    typedef struct _IDT
	{
 		USHORT          Limit;
  		PIDT_DESCRIPTOR Descriptors;
	} IDT, *PIDT;

	static NTSTATUS HookIdtEntry( 
		IN UCHAR DescriptorIndex,
  		IN ULONG_PTR NewHandler,
  		OUT PULONG_PTR OriginalHandler OPTIONAL)
	{
  		PIDT_DESCRIPTOR Descriptor = NULL;
  		IDT Idt;

  		__asm sidt [Idt]

  		Descriptor = &Idt.Descriptors[DescriptorIndex];
 		 *OriginalHandler = (ULONG_PTR)(Descriptor->OffsetLow +
                (Descriptor->OffsetHigh << 16));

 	   Descriptor->OffsetLow  = (USHORT)(NewHandler & 0xffff);
  	   Descriptor->OffsetHigh = (USHORT)((NewHandler >> 16) & 0xffff);
	   __asm lidt [Idt]
	   return STATUS_SUCCESS;
	}

In addition to hooking an individual IDT descriptor, the entire IDT can be hooked by creating a new table and then setting its information using the lidt instruction.

**Category**: Type I; although some portions of the IDT may be legitimately hooked.

**Origin**: The IDT hook has its origins in Interrupt Vector Table (IVT) hooks. In October, 1999, Prasad Dabak et al wrote about IVT hooks[31]. Sadly, they also seemingly failed to cite their sources. It's certain that IVT hooks have existed prior to 1999. The oldest virus citation the authors could find was from 1994, but DOS was released in 1981 and it is likely the first IVT hooks were seen shortly thereafter[7]. A patent that was filed in December, 1985 entitled Dual operating system computer talks about IVT ``relocation'' in a manner that suggests IVT hooking of some form[3].

**Capabilities**: Kernel-mode code execution.

**Covertness**: Detection of IDT hooks is often trivial and is a common practice for rootkit detection tools[32]. 


## GDT/LDT ##

The *Global Descriptor Table (GDT) and Local Descriptor Table (LDT)* are used to store segment descriptors that describe a view of a system's address space1. Segment descriptors include the base address, limit, privilege information, and other flags that are used by the processor when translating a logical address (seg:offset) to a linear address. Segment selectors are integers that are used to indirectly reference individual segment descriptors based on their offset into a given descriptor table. Software makes use of segment selectors through segment registers, such as CS, DS, ES, and so on. More detail about the behavior on segmentation can be found in the x86 and x64 system programming manuals[1].

In Phrack 55, Greg Hoglund described the potential for abusing conforming code segments[19]. A conforming code segment, as opposed to a non-conforming code segment, permits control transfers where CPL is numerically greater than DPL. However, the CPL is not altered as a result of this type of control transfer. As such, effective privileges of the caller are not changed. For this reason, it's unclear how this could be used to access kernel-mode memory due to the fact that page protections would still prevent lesser privileged callers from accessing kernel-mode pages when paging is enabled.

Derek Soeder identified an awesome flaw in 2003 that allowed a user-mode process to create an expand-down segment descriptor in the calling process' LDT[40]. An expand-down segment descriptor inverts the meaning of the limit and base address associated with a segment descriptor. In this way, the limit describes the lower limit and the base address describes the upper limit. The reason this is useful is due to the fact that when kernel-mode routines validate addresses passed in from user-mode, they assume flat segments that start at base address zero. This is the same thing as assuming that a logical address is equivalent to a linear address. However, when expand-down segment descriptors are used, the linear address will reference a memory location that can be in stark contrast to the address that's being validated by kernel-mode. In order to exploit this condition to escalate privileges, all that's necessary is to identify a system service in kernel-mode that will run with escalated privileges and make use of segment selectors provided by user-mode without properly validating them. Derek gives an example of a MOVS instruction in the int 0x2e handler. This trick can be abused in the context of a local kernel-mode backdoor to provide a way for user-mode code to be able to read and write kernel-mode memory.

In addition to abusing specific flaws in the way memory can be referenced through the GDT and LDT, it's also possible to define custom gate descriptors that would make it possible to call code in kernel-mode from user-mode[23]. One particularly useful type of gate descriptor, at least in the context of a backdoor, is a call gate descriptor. The purpose of a call gate is to allow lesser privileged code to call more privileged code in a secure fashion[45]. To abuse this, a backdoor can simply define its own call gate descriptor and then make use of it to run code in the context of the kernel.

**Category**: Type IIa; with the exception of the LDT. The LDT may be better classified as Type II considering it exposes an API to user-mode that allows the creation of custom LDT entries (NtSetLdtEntries).

**Origin**: It's unclear if there were some situational requirements that would be needed in order to abuse the issue described by Greg Hoglund. The flaw identified by Derek Soeder in 2003 was an example of a recurrence of an issue that was found in older versions of other operating systems, such as Linux. For example, a mailing list post made by Morten Welinder to LKML in 1996 describes a fix for what appears to be the same type of issue that was identified by Derek[44]. Creating a custom gate descriptor for use in the context of a backdoor has been used in the past. Greg Hoglund described the use of call gates in the context of a rootkit in 1999[19]

**Capabilities**: In the case of the expand-down segment descriptor, access to kernel-mode data is possible. This can also indirectly lead to kernel-mode code execution, but it would rely on another backdoor technique. If a gate descriptor is abused, direct kernel-mode code execution is possible.

**Covertness**: It is entirely possible to write have code that will detect the addition or alteration of entries in the GDT or each individual process LDT. For example, PatchGuard will currently detect alterations to the GDT. 


## SSDT ##

The *System Service Descriptor Table (SSDT)* is used by the Windows kernel when dispatching system calls. The SSDT itself is exported in kernel-mode through the nt!KeServiceDescriptorTable global variable. This variable contains information relating to system call tables that have been registered with the operating. In contrast to other operating systems, the Windows kernel supports the dynamic registration (nt!KeAddSystemServiceTable) of new system call tables at runtime. The two most common system call tables are those used for native and GDI system calls.

In the context of a local kernel-mode backdoor, system calls represent an obvious target due to the fact that they are implicitly tied to the privilege boundary that exists between user-mode and kernel-mode. The act of hooking a system call handler in kernel-mode makes it possible to expose a privileged backdoor into the kernel using the operating system's well-defined system call interface. Furthermore, hooking system calls makes it possible for the backdoor to alter data that is seen by user-mode and thus potentially hide its presence to some degree.

In practice, system calls can be hooked on Windows using two distinct strategies. The first strategy involves using generic function hooking techniques which are described in 2.1.1. The second strategy involves using the function pointer hooking technique which is described in 2.5. Using the function pointer hooking involves simply altering the function pointer associated with a specific system call index by accessed the system call table which contains the system call that is to be hooked.

The following code shows a very simple illustration of how one might go about hooking a system call in the native system call table on 32-bit versions of Windows: 

    PVOID HookSystemCall( PVOID SystemCallFunction, PVOID HookFunction)
    {
        ULONG SystemCallIndex =
        *(ULONG *)((PCHAR)SystemCallFunction+1);
        PVOID *NativeSystemCallTable =
        KeServiceDescriptorTable[0];
        PVOID OriginalSystemCall =
        NativeSystemCallTable[SystemCallIndex];
        NativeSystemCallTable[SystemCallIndex] = HookFunction;
        return OriginalSystemCall;
    }


**Category**: Type I if prologue hook is used. Type IIa if the function pointer hook is used. The SSDT (both native and GDI) should effectively be considered write-once.

**Origin**: System call hooking has been used extensively for quite some time. Since this technique has become so well-known, its actual origins are unclear. The earliest description the authors could find was from M. B. Jones in a paper from 1993 entitled Interposition agents: Transparently interposing user code at the system interface[27]. Jones explains in his section on related work that he was unable to find any explicit research on the subject prior of agent-based interposition prior to his writing. However, it seems clear that system calls were being hooked in an ad-hoc fashion far in advance of this point. The authors were unable to find many of the papers cited by Jones. Plaguez appears to be one of the first (Jan, 1998) to publicly illustrate the usefulness of system call hooking in Linux with a specific eye toward security in Phrack 52[30].

**Capabilities**: Kernel-mode code execution.

**Considerations**: On certain versions of Windows XP, the SSDT is marked as read-only. This must be taken into account when attempting to write to the SSDT across multiple versions of Windows.

**Covertness**: System call hooks on Windows are very easy to detect. Comparing the in-memory SSDTs with the on-disk versions is one of the most common strategies employed. 



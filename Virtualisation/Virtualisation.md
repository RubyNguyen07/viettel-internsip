
## Rings in Operating Systems

Operating systems have a number of different layers. Each of these layers has its own privileges. We use protection rings term while mentioning this system. Operating systems manage computer resources, like processing time on the CPU and accessing the memory. Since computers run more than one software process, this will bring some issues. Protection rings are one of the key solutions for sharing resources and hardware.

So, what happens is that processes are executed in these protection rings, where each ring has its own access rights to resources. The central ring has the highest privilege. The outer ones have fewer privileges than the inner ones. The figure below represents the protection rings for x86 processor architecture which is one of the common processor architectures.

[Diagram](https://www.baeldung.com/wp-content/uploads/sites/4/2022/03/1_rings3-1024x678.png)


### Ring 0 

The kernel, which is at the heart of the operating system and has access to everything, can access Ring 0. The code that runs here is said to be in kernel mode. Kernel-mode processes have the potential to affect the entire system. If something goes wrong here, the system would most likely crash. Because this ring has direct access to both CPU and system memory.

### Ring 3 

User processes running in user mode have access to Ring 3. Therefore, this is the least privileged ring. This is where we’ll find the majority of our computer applications. Since this ring doesn’t have any access to the CPU or memory, any instructions involving these must be passed to ring 0.

### Ring 1 and Ring 2

On the other hand, rings 1 and ring 2 offer unique advantages that ring 3 lacks. The OS uses ring 1 to interact with the computer’s hardware. This ring would need to run commands such as streaming a video through a camera on our monitor. Instructions that must interact with the system storage, loading, or saving files are stored in ring 2.

These rights are known as input and output permissions because they involve transferring data into and out of working memory, RAM.


## Virtualisation 

Virtualization is a process that allows for more efficient utilization of physical computer hardware and is the foundation of cloud computing.

Virtualization uses software to create an abstraction layer over computer hardware that allows the hardware elements of a single computer—processors, memory, storage and more—to be divided into multiple virtual computers, commonly called virtual machines (VMs). Each VM runs its own operating system (OS) and behaves like an independent computer, even though it is running on just a portion of the actual underlying computer hardware.

It follows that virtualization enables more efficient utilization of physical computer hardware and allows a greater return on an organization’s hardware investment.

Different types of virtualisation: 

[Types of virtualisation](https://media.geeksforgeeks.org/wp-content/uploads/20230324174741/Types-of-Virtualizaton.png)

### Hypervisor 

A hypervisor is a program for creating, running, deleting and resetting virtual machines. Hypervisors have traditionally been split into two classes: type one, or "bare metal" hypervisors that run guest virtual machines directly on a system's hardware, essentially behaving as an operating system. Type two, or "hosted" hypervisors behave more like traditional applications that can be started and stopped like a normal program. In modern systems, this split is less prevalent, particularly with systems like KVM. KVM, short for kernel-based virtual machine, is a part of the Linux kernel that can run virtual machines directly, although you can still use a system running KVM virtual machines as a normal computer itself.


### Virtual machine 

A virtual machine is the emulated equivalent of a computer system that runs on top of another system. Virtual machines may have access to any number of resources: computing power, through hardware-assisted but limited access to the host machine's CPU and memory; one or more physical or virtual disk devices for storage; a virtual or real network inferface; as well as any devices such as video cards, USB devices, or other hardware that are shared with the virtual machine. If the virtual machine is stored on a virtual disk, this is often referred to as a disk image. A disk image may contain the files for a virtual machine to boot, or, it can contain any other specific storage needs.


### Full virtualisation 

This approach, depicted in Figure 5, translates kernel code to replace nonvirtualizable instructions with new sequences of instructions that have the intended effect on the virtual hardware. Meanwhile, user level code is directly executed on the processor for high performance virtualization. Each virtual machine monitor provides each Virtual Machine with all the services of the physical system, including a virtual BIOS, virtual devices and virtualized memory management.

This combination of binary translation and direct execution provides Full Virtualization as the guest OS is fully abstracted (completely decoupled) from the underlying hardware by the virtualization layer. The guest OS is not aware it is being virtualized and requires no modification. Full virtualization is the only option that requires no hardware assist or operating system assist to virtualize sensitive and privileged instructions. The hypervisor translates all operating system instructions on the fly and caches the results for future use, while user level instructions run unmodified at native speed.

Full virtualization offers the best isolation and security for virtual machines, and simplifies migration and portability as the same guest OS instance can run virtualized or on native hardware.

[Diagram](https://media.geeksforgeeks.org/wp-content/uploads/20200519134803/Full-Virualization.png)

### Paravirtualisation 

“Para-“ is an English affix of Greek origin that means "beside," "with," or "alongside.” Given the meaning “alongside virtualization,” paravirtualization refers to communication between the guest OS and the hypervisor to improve performance and efficiency. Paravirtualization, as shown in the Diagram, involves modifying the OS kernel to replace nonvirtualizable instructions with hypercalls that communicate directly with the virtualization layer hypervisor. The hypervisor also provides hypercall interfaces for other critical kernel operations such as memory management, interrupt handling and time keeping.

Paravirtualization is different from full virtualization, where the unmodified OS does not know it is virtualized and sensitive OS calls are trapped using binary translation. The value proposition of paravirtualization is in lower virtualization overhead, but the performance advantage of paravirtualization over full virtualization can vary greatly depending on the workload. As paravirtualization cannot support unmodified operating systems (e.g. Windows 2000/XP), its compatibility and portability is poor.

Paravirtualization can also introduce significant support and maintainability issues in production environments as it requires deep OS kernel modifications. 

[Diagram](https://media.geeksforgeeks.org/wp-content/uploads/20200519134631/Paravirtualization.png)


## Resources 

https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/VMware_paravirtualization.pdf
https://www.geeksforgeeks.org/difference-between-full-virtualization-and-paravirtualization/
https://www.baeldung.com/cs/os-rings
https://www.ibm.com/topics/virtualization
https://opensource.com/resources/virtualization
https://aws.amazon.com/what-is/virtualization/
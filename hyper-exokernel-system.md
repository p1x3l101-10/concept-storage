Booting:

1. Bootloader will load two things into memory
  - The exokernel
  - The IIS (initial init system), this is a file similar to the kernel, but will run in userspace

2. Exokernel will be started with knowlage of two things
  1. Where the IIS starts
  2. How long the IIS is

3. The exokernel will set up everything that it needs to do for exokernel things
  - Hardware multiplexing mainly
  - A small buffer for modules to communicate over
  - A table of libOSs that are registered
    - Initialized with NONE for the IIS
      - The kernel will go through the process of registering a libOS (an empty one) for the IIS, and then start the IIS with it
    - Once NONE stops being used, it can be dropped from the table as it isnt special other than it is just an empty libOS
    - The NONE libOS should also be initialized with a capability for everything so that it can be passed on to the main system

4. The IIS will be started as PID 1 under libOS NONE

5. IIS will then do some basic setup
  - Load a basic driver bundle containing the device drivers to start the next stage
  - Load an initrd
  - Find the libOS kernel module that is embedded in the initrd
  - Tell the kernel to load it, and copy all the capabilites to it as well because we are transfering control, not spawning a child process
  - Tell the kernel to execute a program under a different libOS (the one we just loaded)
  - Once control is transferred, the kernel must track the change of libOS, and drop NONE from the table

6. Typical-ish unix boot after this

---

If a process wants to start another libOS, here is what needs to be done:

1. The libOS needs to be loaded
  - A timer must be set for how long the kernel will wait for a process to be started on it before unloading the module
  - The exokernel must make sure that the libOS id is available, otherwise return some failure (e.g., tried to load "Linux" when "Linux" exists already)
  - If this libOS should be loaded with the real process table and shared communications with other libOSs, or should be completely isolated
  - The capabilities that the caller wants to pass on to the new libOS (disk sector ownership/permissions, shared memory to communicate with a parent process, etc.)
    - Because the exokernel does not know about filesystems, the actual semantics of what will be passed should be negociated with the libOS that is managing these things
      - Example:
        - Process: I want to give this new libOS rw access to this file, the new libOS should support the current filesystem. 
        - libOS: Here is a capability token that has ro for these disk blocks (these blocks contain filesystem metadata in this example), and rw for the blocks containing that file.
      - Example 2:
        - Process: I want to give this new libOS rw access to this file, but the new libOS does not support the current filesystem. You have a driver for <x> filesystem that is supported by it though
        - libOS: Here is a capability for this space of RAM that contains a ramdisk with that file and filesystem, make sure to tell it that this file is in a ramdisk
      - Example 3:
        - Process: I want to give this new libOS rw access to this file, the new libOS should support the current filesystem.
        - libOS: No, you do not have access to that file, so the new libOS should not either
        - Process should then abort attempt to start the new libOS

2. The process must be started using a special syscall to the exokernel itself, not the libOS
  - This would consist of a couple of parts:
    - The actual memory address of the program (kernel does not know about filesystems)
    - The id of the new libOS to start the program under
    - The PID to start under
      - If running in an isolated libOS, the kernel should return an error if the PID is not 1

Internally, when a libOS starts a new process, it also needs to load the process into memory (like a normal kernel), but it does not need to start a new libOS, it should just reuse its own id in this case.

---

Allow virtualizing hardware devices in the exokernel so that if a libOS does not want to share the ethernet, another libOS has an interface to the internet that it can use without caring that it is a fake ethernet
  - Drivers may be needed for this special device, like all other devices virtual or not

---

A major thing that makes this different than a textbook exokernel is that libOSs are loaded as kernel modules, and not as normal libraries. However, the exokernel itself still handles hardware multiplexing, the modules are more in between user mode and kernel mode, where they have more control than a user process, but they still need to abide by the exokernel's rules when it comes to actually accessing hardware (i.e. needs the right capability token). The exokernel also handles bare-bones virtual process tables with the purpose of tracking which process belongs to which libOS. Also, although not needed in most cases, a process could call most exokernel syscalls directly (using a different interrupt, such as 0x10 instead of 0x80). A process could also load it's own libOS module for a specific task (such as a webserver loading an optimized module instead of a more general purpose one), but it would still need capabilities from it's parent libOS to access hardware, or it might just need to directly talk to it's parent libOS using something like shared memory for things like opening a port. The exokernel itself is not aware of this, all it sees is that two libOSs have read/write on some memory, and one of them is controlling some networking stuff. 

A process does not actually care about tokens except when it is spawning a new libOS. Although every process is required to be under a libOS, because processes can reparent to another libOS (by way of executing a stage 2) they can specify a different interface for the system that they can control.

There is also a minor abstraction built into the kernel where a libOS can create a virtual device that other libOSs can use (say ethernet or a virtual disk), and the exokernel would support granting capability tokens for these virtual devices too. If you want a hurd style networking server, it would need to have virtual ethernet devices that other modules could use as if they were just a funky etherneth port, and the libOS in charge of that device could do smarter multiplexing that the exokernel couldn't dream of doing.

The kernel should also allow processes to check what capabilities that it's parent libOS has without letting it spawn a new libOS with the same capabilities with it's parent using this method. For example, if an anticheat wants to make sure that it has its own section of memory that only it's process controls, then it can. However a malicious process should not be able to abuse this feature to spawn a hostile libOS with full control over the system using this method (ideally, this should not be possible).
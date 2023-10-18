### P1 The xv6 Kernel-1 Intro and Overview

(00:00) This is the first in a series of videos on the xv6 operating system kernel.
(00:07) This is a very short but very sweet Unix-like operating system that's used
(00:12) for educational purposes. It was developed at MIT and used to
(00:16) other places as well. This is
(00:20) an operating system that's used by students primarily in an operating
(00:24) system course, and there are
(00:27) two implementations of this kernel. One for the x86 architecture, and one for
(00:35) the RISC-V architecture. In this series of videos, I'm going to be
(00:39) talking about the RISC-V version. The RISC-V version that is used is a
(00:45) 64-bit processor and whether using the x86 version or the
(00:50) RISC-V version, you're probably going to be using it in
(00:55) an emulated fashion, using an emulator like QEMU.
(01:00) Most likely you don't have a spare computer sitting around.
(01:04) Probably a spare RISC-V
(01:07) processor is even less likely. So, instead you'll be running the
(01:12) operating system if you choose to run it under an emulator.
(01:16) But in any case, it's meant to run on a bear machine,
(01:20) and in fact it's a multi-core operating system, so
(01:25) QEMU is capable of emulating multi-core systems.
(01:30) As I said, it's short and very sweet. It's only about 6000 lines of code.
(01:36) Most of it is written in the c programming language
(01:40) with maybe about 300 lines in assembly language.
(01:45) In this video series, what I'm going to do
(01:48) is do a walkthrough of more or less all of
(01:52) the code to give you an idea of what's going on with it.
(01:56) The code is very simple and well-written and clean code
(02:00) and I've read a lot of c code and written a lot
(02:04) of c code and I'm still learning new coding techniques and I think this is an
(02:08) example that is worth studying.
(02:13) In addition, of course, as an operating system
(02:16) kernel, it illustrates some of the basic concepts that you'd be learning in an
(02:21) operating systems course. So, that's another good reason to study this code
(02:25) in detail. For this video series,
(02:29) I'm not going to assume that you have any
(02:32) knowledge of the RISC-V instruction set architecture,
(02:37) but I will assume that you have had some assembly language coding.
(02:41) I intend to walk through the assembly language
(02:44) instructions line by line. So, I will hold your hand
(02:49) there, so don't worry. Have you had an operating system class?
(02:54) Maybe you're in an operating system class right now that is using the
(02:58) xv6 system. In any case, I've taught a few
(03:01) operating system classes and I'll probably
(03:05) go over some of the concepts as we encounter them.
(03:08) This is going to be a long series, so buckle your seat belts. It's not for the
(03:13) faint of heart. I'm assuming that you've got some brains
(03:17) and furthermore, I'm assuming that you're interested in this particular system and
(03:22) can focus for the amount of time that it's going to be
(03:26) for this video series. So let's look at some of the features
(03:30) that the kernel has. It's got
(03:35) processes. These processes run in their own virtual
(03:39) address spaces. so
(03:42) There are page tables for, a page table for each address space to support the
(03:47) virtual address spaces. The operating system supports
(03:52) files, Unix-like files and the directory hierarchy.
(03:57) You can pipe data from one program to another program.
(04:03) Of course, there's a timer interrupt, so there is multitasking. The the various
(04:10) processes are running in parallel with time-slicing.
(04:14) There are 21 system calls that are implemented in
(04:18) xv6. This is not a lot. The
(04:23) production Unix systems have more like 300 system calls.
(04:28) maybe 500 system calls, but this is enough
(04:32) to give you the core ideas of Unix. There are a number of user programs that
(04:39) are supplied with this kernel and
(04:43) these can illustrate the capabilities of this operating
(04:48) system. The operating system can run a simple
(04:51) shell program. In fact, I made a video that talks about
(04:54) the shell program in detail. You can look for that if you want. [The videos is: Shell Code Explained](https://www.youtube.com/playlist?list=PLbtzT1TYeoMhF4hcpEiCsOeN13zqrzBJq)
(04:58) Other common Unix programs: cat, echo grep
(05:04) kill, which is used to terminate a process,
(05:07) ln, which is used to create a hard link from one file to another
(05:12) and ls, which is used to list out the
(05:16) contents of a directory. You can create directories.
(05:19) You can remove files and
(05:22) wc is for counting the words in a file as well as the characters.
(05:29) So, all in all, I think that this can really be
(05:32) considered a true Unix system although it's pretty short and simple.
(05:38) There's a lot that's missing. Okay, definitely,
(05:42) all the complexity of a real operating system is not there.
(05:46) A real operating system like linux may have, you know, as much as a hundred times
(05:50) as much code, and you know, we're talking a million
(05:54) lines of the kernel, and you know, when you add the device drivers in,
(05:58) you can go up to many millions of lines of code. 
(06:03) This is just too much for any human
(06:06) to study ,really, and if you want to find out how kernels work, this is really the
(06:11) operating system for you, but there are some things that are
(06:14) missing from your typical Unix or linux system.
(06:19) There are no user IDs and no login sequence, no
(06:24) verification. There are no protection bits associated
(06:29) with files, you know, the read, write, execute
(06:31) protections, that's not here. The "mount" command is just not available,
(06:37) So you just have one file system.
(06:41) In a real system, the virtual address spaces can be paged
(06:45) out to disk so that you can run more processes than will fit in physical
(06:52) main memory. That's not present
(06:55) in xv6. There's no
(06:59) support for networks, no sockets or anything like that.
(07:04) In fact, there's no way for processes to communicate
(07:07) or synchronize amongst themselves. There are
(07:12) two device drivers, but a real operating system, a real world operating system is
(07:18) going to have many more device drivers to support all kinds of different bits
(07:22) of hardware that you might find, and lastly there is only a limited
(07:28) amount of user code. I listed the
(07:32) approximately 10 programs that are
(07:36) distributed with it. But a real usable linux or using unix system is
(07:41) going to have lots and lots of apps. Let's go over some of the
(07:47) system calls that are present. I'll go over these in more detail later when we
(07:51) encounter them. but i just want to kind of list them out here, so you could see
(07:55) what we've got.
(07:58) fork(), these are familiar from any Unix or
(08:02) linux system. The parameters are slightly different.
(08:06) In some cases but the idea is there
(08:11) in concept anyway
(08:14) fork() is used to create a new process, wait() is used to
(08:19) wait for a child process to terminate, exit() is for terminating a process, pipe()
(08:26) is for creating pipes and then we've got open()
(08:30) close(), read() and write() for dealing with files.
(08:36) We've got kill() to terminate a process. We've got exec(),
(08:41) which is passed a file name and we'll read in that
(08:45) file, presumably it's an executable file, and we'll load it into
(08:49) memory, creating a new virtual address space and execute it. We can
(08:55) make inodes, we can
(08:58) create links, hard links and we can remove hard links
(09:03) and unlink files, thereby possibly removing
(09:07) them if it's the last link. We can get information about files.
(09:12) We can change directory, so we do have a notion of the current working
(09:16) directory. dup()) is used for copying file
(09:19) descriptors. Now we can
(09:22) get program id, sorry, the process id for the current process.
(09:28) We can grow the heap,
(09:30) so, that's this function here, this system call here.
(09:34) And we can put a process to sleep for a while. We can also see how long the
(09:39) kernel has been running. So, in the next videos and in this series,
(09:43) I'll be going through the code in quite a bit more detail, but i just want to
(09:47) start with giving you an idea of what this operating system kernel has
(09:52) and what its capabilities are, so you can determine whether you want to
(09:57) make the commitment for watching these videos. As I said,
(10:01) this series is not for the faint of heart. It's not for the amateurs, it's for
(10:06) people who really want to look at an operating system kernel in
(10:09) detail and understand a rather large but not
(10:14) too large body of code. Okay, let's get started with the next
(10:18) video.
---
layout: page
title: WSL, SUA, and YOU
---

Microsoft introduced what the press & vernacular are calling alternately "Bash on Windows" or "Ubuntu on Windows" in the Insider Preview (ie, beta) builds of Windows 10 officially titled "Windows Subsystem for Linux" or WSL. Despite the press excitement, a POSIX-compatible subsystem in Windows isn't especially new and can be traced back as far as the Interix integration in to NT4 back in 1999, but the mechanism of WSL is not related to these previous efforts except in general concept, and is at the very least an interesting new engineering solution.

The intent of this post is to document my explorations as a very knowledgable UNIX user, admin & hacker in to the workings, rough edges & caveats of the WSL subsystem. I do not have much experience in Windows, as this is the first time I've used it since roughly Windows 2000 SP2 was released. I do have some cursory knowledge of the inner workings of Windows but I'm not what you would call a "Power User." My daily driver is macOS, precisely because it is UNIX with a much nicer UI. On the other side of that coin, I'd happily use Windows if the Linux/UNIX subsystem became a first class citizen.

## How it works

Unlike Interix/SFU/SUA, which is essentially a libc and a compiler that understands Windows, the main piece of WSL is a kernel module that interpolates syscalls from userland processes, re-marshalls the arguments to fit Windows calling standards, and then passes the data on to a corresponding Windows syscall if it exists or reimplements the Linux handler if it does not. This is remarkably similar to the way Solaris/Illumos handles lxrun / lx branded zones, or how FreeBSD and others provide binary compatibility. The difference between those implementations and the WSL implementation is that while BSD & Solaris/Illumos are POSIX compatbile operating systems with SysV calling conventions, Windows is neither. So while other UNIX operating systems can get away with more or less remapping the syscall number, Windows has to completely unmarshall, interpret & pass on the function arguments to Windows kernel calls or reimplement some calls entirely.

WSL also provides a sort of pseudo VFS-layer (ie, filesystem operations) which allows it to work around deficiencies in the Windows filesystem model on the one hand (for instance nesting mounts), and provide pseudo-filesystems such as procfs and tmpfs on the other.

In practice, for the vast majority of cases Linux binaries should see nothing different from a standard Ubuntu install. They hook in to the same glibc they're used to, calling conventions remain the same all the way up to the kernel (whose job it is to "do the right thing".) The way WSL implements this is as a full Ubuntu install living in the Windows filesystem.

In terms of the system itself, the guest filesystem is stored in C:\\Users\\<username>\\AppData\\Local\\lxss. The files are available through Windows and the Windows files are available in the guest under /mnt/c. There is a vague warning from Microsoft that because of some artifact of the VFS module's caching, you should not modify the Linux files from Windows directly but the opposite effect is not true (ie, you can modify the Windows mount from Linux.) The WSL process is launched as bash.exe. Running it through various debuggers (IDA, Olly) it appears that this process is bash, statically linked with a libc (it contains private libc functions) and some Win32 wrapper for the actual console code (almost certainly cmd.exe based). 

## How it doesn't work

SUA/SFU, being just a C standard library but ultimately living in the same container space as the host Windows, acts more or less like Windows applications as far as permissions, process tree, naming etc. are concerned. WSL on the other hand does not seem to do any of this. The Bash process, bash.exe, does not seem to be, as you would expect, a terminal emulator & bash process that opens a pty (since there are no pty's in Windows); it appears to be the controlling process for the whole WSL instance. This would imply that daemons don't appear to work and once the bash process is quit (by closing the window or logging out) any sub-processes are also terminated. No screen or tmux, or mysql or Apache (both of which have Windows ports anyway.) It also is the case that users, permissions and the likes do not work as you would expect them to.


## Surprises

The first surprise is that once you close bash.exe everything terminates. It would suggest that either init is tied to the bash.exe process, or that Windows is launching & tearing down a daemon of some sort in the background. There are no boot messages when you first open bash.exe, but unclear what that implication might be.

`who(1)`, `uptime(1)`, `last(1)` and anything else that normally reads the utmpx database doesn't work. It appears that WSL is not recording login information (and likely does not even have a concept of it.) `write(1)` and `wall(1)` also do not work as it appears there the concept of pty's in WSL are limited. `ps(1)` does show other processes launched by other instances of WSL (caveat: by the same user. I did not test multi-Windows user) each process is attached to a tty ( the first bash.exe instance seems to start at tty3 ) but UNIX signalling to those tty's (ie, `write(1)` with the tty argument) also did not seem to work, but passing `tmux` between instances does. Windows seems to be keeping process state somewhere and tearing it down when you close all the bash processes. Weirdly, despite the other problems with pty's tmux was successful in creating new windows.

The system does not include an X server. Xming is reported [1](https://www.slightfuture.com/how-to/x-on-wsl) to work for toy examples (Xeyes, Xclock) but nothing more complicated and especially nothing that relies on DBus. The theory is that it is an incomplete implementation of pipes. Debugging crashes in X applications (eg, Firefox) also reportedly  did not work because of gdb's failure to correctly handle threads. I have no evidence to support or contradict this theory. As the system matures I expect this aspect to resolve itself.

Because the controlling processes are launched by the currently logged in user, and every user is given their own fresh "install" of Ubuntu, UNIX users are more or less meaningless. The user who launched the bash.exe process is root, and the Ubuntu install creates it's own /etc/passwd and such. When interacting with the host OS files are created by the host's user rather than any UNIX usernames you may have created. The guest filesystem from the view of Windows is again all owned by the host's username. /etc/shadow is owned by my Windows user in Windows and root in Linux. The install directory ( %LocalAppData%\\lxss\\rootfs ) is not available through the /mnt/c mountpoint to any UNIX user so there is no risk of underprivileged users going through the mountpoint to pull out password data. New files on the guest filesystem are created, regardless of the UNIX user, as root on the guest system and the local user on the host and every operation on the host filesystem is as root (but again, as the logged in Windows user not as Administrator.)

## Conclusions


Essentially this is not a UNIX/Linux subsystem. Most accurately, it is an OS-level virtualized instance (like jails in FreeBSD or zones in Solaris/Illumos) that is an approximation of Linux. Many things do not currently work, but many do. Depending on your reasons for using it, it may be sufficient.

I hesitate to hint at disparaging a system still in beta and by no means do I mean to do that. I'm simply trying to examine the system with the lens of architectural caveats.

## Suggestions

Were I on the WSL team and given infinite resources I might consider writing an nsswitch module to interrogate the host OS for user data. That would allow the WSL system to run as a daemon on Windows & allow daemons launched in Linux to limit their permissions on the /mnt/c mountpoint. Linux does contain the ability to understand NTFS ACL's as it both has an NTFS module and NFSv4 module. The second would be to launch the instance at boot but I suspect this will show up in the server builds in time anyway, once the pty subsystems improve.

## Further work

I briefly attempted to write a UNIX program to launch Windows applications via shellcode, which resulted in a segfault. Given how the system works, unless Microsoft has specifically guarded against it, it should be theoretically possible to make Windows syscalls from within WSL. I am not knowledgable enough about Windows kernel calling to do this at the moment.

[1] [https://www.slightfuture.com/how-to/x-on-wsl](https://www.slightfuture.com/how-to/x-on-wsl)

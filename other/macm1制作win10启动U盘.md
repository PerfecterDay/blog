
FAQs
Can I format USB to exFAT and burn ISO to USB without split?

No. Although exFAT does not have 4GB file size limitation, the USB drive will not be seen a bootable device by motherboard, which only supports FAT2 and NTFS.


The latest file size of Windows 10 20H2 is 5.73 GB. And install.wim file is larger than 4GB.

WIM is an acronym for Windows Imaging; It is an imaging format that allows a single disk image to be used on multiple computer platforms. WIM is usually used to manage files like updates, drivers, and system component files without having to reboot the operating system image.

You won’t receive any error during the ISO burning process but Windows 10 installation is unable to complete. You will receive ‘ A media driver your computer needs is missing’ error at the middle of Windows 10 install.

This means you cannot directly burn this ISO to USB because FAT32 file system is only capable of storing a single file less than 4GB. And FAT32 is the only working option for creating Windows bootable USB drive.

How about NTFS? NTFS is a patent-protected file system owned by Microsoft. Apple’s macOS does not support NTFS natively so it is impossible to burn ISO to a NTFS-formatted USB on Mac. That’s the problem! And a lot of users don’t know those details and wasted hours without any success.

Simply put, you can only make a bootable Windows USB on Mac with USB formatted to FAT32.

Currently, there are two workaround to fix this issue. The first way is to split install.wim file into small pieces and copy them to USB. This can be only achieved in Terminal with text commands. Another way is to download a small size of Windows 10 ISO and burn it to USB directly on Mac, which is much simpler than using Terminal command.



1. ls /Volumes
2. 双击下载的ISO文件加载
3. rsync -vha --exclude=sources/install.wim /Volumes/CCCOMA_X64FRE_ZH-CN_DV9/* /Volumes/WIN10
4. wimlib-imagex split /Volumes/CCCOMA_X64FRE_ZH-CN_DV9/sources/install.wim /Volumes/WIN10/sources/install.swm 3500


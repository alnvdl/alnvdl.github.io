---
layout: post
title: "Windows corrupts Debian install media"
date: "2018-02-14 14:00:00 -0200"
excerpt: A tale of two operating systems and one really silly issue
---

A few weeks ago I downloaded the Debian net install image, [verified it](https://www.debian.org/CD/verify), flashed it to a flash drive (using `dd`), verified again after flashing (`head ... | sha512sum`). The checksum was correct, and I proceeded to install Debian as usual.

While trying to boot from the flash drive, I ended up missing the right timing and Windows 10 started up instead. I rebooted and installed Debian after managing to boot from the flash drive.

Call me paranoid, but that accidental Windows start up made me wonder: what if another OS could get in the way and modify the Linux install process? What about the UEFI firmware, couldn't it do something as well? So I decided to verify that the flash drive was still intact.

Well, guess what, it had happened. I calculated the checksum just as I had done before, and the results were different. I took some binary diffs (`cmp`), tried to `hexdump` parts of the content and found some strings with that typical CamelCaseEverywhere, Windows-like feeling: "SystemVolumeInformation". With that clue, but without the willingness to actually investigate every byte, I decided to try to reproduce the problem.

I spent a few hours manually selecting entries from the UEFI boot menu, changing the boot order with the flash drive inserted, booting to Windows, removing the USB drive during the boot process, or after Windows had started... In a few occasions I could get it to change the checksum when starting Windows with the flash drive inserted, but not always. Then one time, almost giving up, I decided to give Windows a few minutes with the flash drive inserted.

And that did the trick. These are the steps to reproduce the issue:

1. Write your Debian install image to a writeable drive, and verify that it is correctly written
2. Boot into Windows and wait for approximately 5 minutes (don't login, just wait)
3. Verify the checksum of your writeable install media again, and there you have your "corrupted" install media

Since my Windows install was clean, I didn't really suspect of malicious software. It was probably just some weird behavior. Then I decided to do something I should have done from the beginning: look into the partition table. There I learned that the Debian install media has two partitions:

1. The main partition with all the install files
2. A boot partition, contained within partition 1 (I didn't even know that was a thing). This partition actually maps to a file called `/boot/grub/efi.img` in the main partition.

If you mount and modify that boot partition (which is a FAT partition by the way), you end up modifying the `efi.img` file, and the checksum will fail.

When I mounted that boot partition, I saw a folder named `System Volume Information`, with a file named `IndexerVolumeGuid` within. The content of this file was an apparently random ID. So that was it: **Windows is trying to index a filesystem for whatever reason, before I even login**. In the process, it ends up changing a file beloging to the Debian install media, which may cause verification tools to say the install media is corrupted. The directory and file Windows creates are not that well documented, but there's [some information available](https://support.microsoft.com/en-us/help/309531/how-to-gain-access-to-the-system-volume-information-folder).

In this case, Linux is just a victim of the not-so-nice practices of another OS' eagerness to index things. For Windows, it's certainly a poor choice regarding file system indexing behavior. In the worse case scenario, a security flaw in whichever code is indexing that flash drive could be exploited by simply pluging a flash drive to a computer, no login required. I didn't submit this to the Microsoft security team because, as they say in their [10 Immutable Laws of Security, Law #3](https://technet.microsoft.com/library/cc722487.aspx#EIAA), if someone has physical access to plug flash drives in your computer, they probably could do much worse (even a sledgehammer would do, as one of their examples says :)

How could this be addressed? Well, Windows could be a nicer citizen of OS-land and avoid trespassing the properties of others. Alternatively, if there were a way to proactively disable Windows indexing (something like a `robots.txt` file), the Debian install media could include it to avoid this undesirable side effect.

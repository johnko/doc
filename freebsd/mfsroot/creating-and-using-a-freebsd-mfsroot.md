#Creating and Using a FreeBSD mfsroot

Retrieved on 2014-07-25 17:10:49 UTC+0000 from http://www.michaelmiranda.org/software-development/articles/hawaiisidentitytheftlaw

posted Dec 31, 2009, 4:19 PM by Michael Miranda - A [ updated Jan 30, 2010, 5:25 PM ]

by Michael J. A. Miranda, December 28, 2007

##Purpose of an mfsroot

FreeBSD has a means to run with a file system that resides fully in memory (RAM). The primary benefits of such a system are:

- By running in memory, the operating system operates at a faster speed.
- Corruption of the operating system's filesystem in memory can be "repaired" by simply rebooting the machine with the original filesystem.
- You can create an "embedded system", "firmware" or "appliance" type of machine that can be updated on the fly by simply replacing the filesystem file (mfsroot file) that gets loaded and executed in memory (similar to network routers and switches)
- You can create several versions of the operating system's filesystem and switch them out easily as necessary

##What does it look like on the hard drive

One of the first hurdles to successfully completing a project like this is understanding what the final product will look like. In this case, we are going to end up with a bootable hard drive where the filesystem is contained in only one file. For FreeBSD, this means that the actual bootable hard drive (or other boot device such as usb drive or compact flash card) filesystem will look like this on the bootable partition:

```
/boot
  /kernel
    /kernel
  /loader.conf
  /[...other standard boot files]
/mfsroot.gz
```

That is it.

The first question is: Where are all the operating system files and directories? Where are `/usr /usr/local/ /etc /bin /sbin ....etc.`?

The answer is: All operating system files are in the `mfsroot` file (`mfsroot.gz` compressed).

The mfsroot file contains the entire file system excluding the `/boot` directory and associated files.

##Creating an mfsroot file

###Basic Steps

1. Create a filesystem that contains all of the FreeBSD components, programs and ports that you require.
2. Create an mfsroot image file of a size that can contain the filesystem.
3. Mount the mfsroot file as a memory disk so that it is available and operational as a disk device.
4. Copy the filesystem into the mounted mfsroot memory disk.
5. Unmount the mfsroot memory disk and close the mfsroot image file.
6. Compress the mfsroot image file using gzip.

###Short of the Long

####Create a filesytem

The best way to create and test the filesystem that you are going to use for the mfsroot is by utilizing FreeBSD's *jail* system. Use the jail system to install another instance of FreeBSD on your current FreeBSD machine. *chroot* into the jailed system and install all the necessary packages. Test the filesystem until it meets your requirements. (Google "FreeBSD jail" for more information)

####Create an mfsroot image file

- [output_file] = name of the mfsroot file (usually *mfsroot*)
- [block_size] = size of the blocks of memory in bytes
- [number_of_secotrs] = number of blocks of memory that is at least as large as the size of the filesystem

To determine [block_size] and [number_of_sectors] you must first know how big your filesystem is. Execute `du -h [path_to_jail_filesytem]` to obtain the number of megabytes that the filesystem occupies. Next, use the following conversion table.

Number of Megabytes * 1024 = Number of Kilobytes
Number of Kilobytes * 1024 = Number of Bytes
[number_sectors] = Number of Bytes / [block_size]

Decide on a [block_size]. Usually, 512 bytes is used.

For example, a 120 megabyte file system = 120 X 122,880 kilobytes. (122,880 * 1024) / 512 block size = 245,760 sectors.

```
dd if=/dev/zero of=[output_file] bs=[block_size] count=[number_of_sectors]
```

####Mounting the mfsroot image file

Prior to copying the filesystem into the mfsroot image file, you must mount the image file to make it accessible like a disk device. This can be accomplished by creating a memory disk in RAM and mounting it.

```
mdconfig -a -t vnode -f [output_file]
```

Example:

```
mdconfig -a -t vnode -f mfsroot
```

This command will return a memory disk device name (i.e. md0, md1). This is the device you need to mount in order to access the mfsroot image file as a disk

```
mount /dev/md0 /mnt
```

The above command essentially mounts the mfsroot image file to the `/mnt` directory. All you need to do now is copy your filesystem to the `/mnt` directory to have it added to the mfsroot image file.

####Copying the filesystem to the mfsroot mounted image

When you copy the filesystem, you will want to preserve the owner and permission settings of all the files. The best way to do this is by using `tar`.

```
tar cf - [filesystem_root] | tar -C [target_directory] -xfp -
```

Example:

```
tar cf - /usr/jail/myfilesystem | tar -C /mnt -xfp -
```

####Umounting the mfsroot image file

Close the mfsroot image file by simply unmounting the memory disk device.

```
umount /dev/md0
```

####Compress the mfsroot image file

```
gzip -9 [output_file]
```

Example:

```
gzip -9 mfsroot
```

##MFSROOT Filesystem Size

The size of your mfsroot file system (the filesystem you built using a `jail` process) will determine whether you need to make any modifications to the default kernel. First of all, you must consider that the mfsroot filesystem will be loaded into memory upon boot up. Therefore, you need to make sure the destination machine has sufficient memory to hold the filesystem and run the applications. For an embedded/firmware/appliance type of system you would prefer to have a filesystem that is as small as possible. However, if your memory availability and resource needs allow, almost any size could be accommodated. If you build and use the GENERIC kernel, your mfsroot filesystem should be no larger than 100 megabytes to allow it to be loaded by the kernel. If it is larger than that, your the mfsroot filesystem will not load and your machine will continually cycle reboots. In which case, you will need to modify the kernel options to accommodate the larger mfsroot file size.

##Build a Kernel

The kernel and the other files required to boot the system is located outside of the `mfsroot` file. The best practice is to build a custom kernel to use with your mfsroot-based FreeBSD system. For instructions on how to build a kernel, go to the following link:

http://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/kernelconfig-building.html

**IMPORTANT NOTE: DO NOT INSTALL THE KERNEL ON YOUR SYSTEM. Simply use this process to build and compile a kernel to use for the mfsroot-based FreeBSD system you are building.**

###Large MFSROOT Filesystems Require Kernel Source Code Changes

Although the goal when builing an embedded/firmware/appliance system is to reduce the footprint of the filesystem in memory, sometimes business requirements dictate a filesystem size larger than what is allowed by the default kernel configuration. To allow the kernel to loadlarger `mfsroot` filesystems, you will need to make the following changes to the kernal source code:

- Find and open for editing the `/usr/src/syst/i386/include/pmap.h` file.
- Find the following variable: `NKPT` and increase its value.
- Find the following variable: `KVA_PAGES` and increase its value.
- Save and exit the `pmap.h` file and rebuild the kernel as described above.

The most obvious question is: How much do I increase the value of those variables? See the formula below

####NKPT and KVA_PAGES Formulas

To Be Developed

##Using an mfsroot file

See https://github.com/mmatuska/mfsbsd

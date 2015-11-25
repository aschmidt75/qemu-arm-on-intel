# qemu-arm-on-intel
Sample ARM-VM setup on Ubuntu/Intel using Qemu


## What and why

This is a small walk-through of how to set up an Ubuntu Wily ARM-installation
on a Ubuntu Intel host.

I was working on a number of tools, which are also aimed to be run on a
RaspberryPi or similar ARM-based Linux Devices. There was need to integrate
the building and testing of these tools in a continuous delivery build chain,
with testing steps. Testing can also be done on i386/x86_64 intel hardware, but in the
end - when delivering a product to a customer - we wanted to have our code tested on ARM-installations.

Of course this can be reached by using RaspberryPi's or something similar such
as CubieTrucks, having them run as a server-like standby and integrating them in the
build pipeline. However this does not work well when using compute resources in
some other locations or even cloud-based services.

So, a Qemu-based emulation of an ARM environment seemed useful to me.

This article is based on a wiki page of Ubuntu.com, showing the basics to set up
Ubuntu Saucy. It needed some extra care when switching to a newer version,
but nevertheless it worked, and the original article by Paolo Pisati was really helpful:

 * (https://wiki.ubuntu.com/Kernel/Dev/QemuARMVexpress)[https://wiki.ubuntu.com/Kernel/Dev/QemuARMVexpress]

## How

I ran this example on a trusty64 notebook:

```bash
~# uname -a
Linux c018-ThinkPad-W530 3.19.0-33-generic #38~14.04.1-Ubuntu SMP Fri Nov 6 18:17:28 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

~# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 14.04.3 LTS
Release:	14.04
Codename:	trusty
```

First up i needed to install some extra packages on the host:

```bash
$ sudo apt-get install qemu-user-static qemu-system-arm debootstrap
```

debootstrap installs a fresh system into a new directory, the others are
CPU Emulators for ARM from Qemu.

All of the following runs locally in a directory, `vexpress` (or whatever you like).
The OS installation will reside in a subdirectory `qemu-img` therein:

```bash
~$ mkdir -p vexpress/qemu-img
~$ cd vexpress/
~/vexpress$
```

Next up, we need an - initially empty - image for our OS installation. `dd`
creates us a 4GB, zeroed-out image file, `losetup -f` will look for a free /dev/loopX-
device and attach the image to it. Then we create an ext4 file system inside
and mount it to our image subdirectory, so we can make changes to it:

```bash
~/vexpress$ dd if=/dev/zero of=./vexpress-4G.img bs=4M count=1024
1024+0 Datensätze ein
1024+0 Datensätze aus
4294967296 Bytes (4,3 GB) kopiert, 24,8169 s, 173 MB/s

~/vexpress$ sudo losetup -f ./vexpress-4G.img

~/vexpress$ sudo mkfs.ext4 /dev/loop0
mke2fs 1.42.9 (4-Feb-2014)
Dateisystem-Label=
(...)

~/vexpress$ sudo mount /dev/loop0 qemu-img

$ mount
(...)
/dev/loop0 on /home/c018/vexpress/qemu-img type ext4 (rw)

$ ls -al qemu-img/
insgesamt 24
drwxr-xr-x 3 root root  4096 Nov 19 12:46 .
drwxrwxr-x 3 c018 c018  4096 Nov 19 12:44 ..
drwx------ 2 root root 16384 Nov 19 12:46 lost+found

```

Now comes in `qemu-debootstrap` which is a wrapper to `debootstrap` that does
additional things for all tof this to work inside a Qemu process. See manpage
for more details. We install Wily into `qemu-img` subdir:

```bash
~/vexpress$ sudo qemu-debootstrap --arch armhf wily qemu-img
I: Running command: debootstrap --arch armhf --foreign wily qemu-img
I: Retrieving Release
I: Retrieving Release.gpg
(.... tons of logs ....)
```

The sub directory now contains an OS installation base:

```bash
$ ls -al qemu-img/
insgesamt 96
drwxr-xr-x 21 root root  4096 Nov 19 13:17 .
drwxrwxr-x  3 c018 c018  4096 Nov 19 12:44 ..
drwxr-xr-x  2 root root  4096 Nov 19 13:13 bin
drwxr-xr-x  2 root root  4096 Okt 19 11:16 boot
(...)
```

We need a binary, `qemu-arm-static` to be present in the chroot:

```bash
~/vexpress$ sudo cp `which qemu-arm-static` qemu-img/usr/sbin/
```

The next block is about some basic configuration of the OS. This is done
by chroot-ing into the sub directory.

 * configure a /dev/ttyAMA0
 * adding a repository to apt sources
 * running an update
 * configuring network
 * setting root password

```bash
$ sudo chroot qemu-img
/# echo '# tty1 - getty
#
# This service maintains a getty on tty1 from the point the system is
# started until it is shut down again.

start on stopped rc RUNLEVEL=[2345] and (
            not-container or
            container CONTAINER=lxc or
            container CONTAINER=lxc-libvirt)

stop on runlevel [!2345]

respawn
exec /sbin/getty -8 38400 ttyAMA0' >/etc/init/ttyAMA0.conf

/# echo "deb http://ports.ubuntu.com wily main restricted multiverse universe" >/etc/apt/sources.list

/# apt-get update
Get:1 http://ports.ubuntu.com wily InRelease [218 kB]
Get:2 http://ports.ubuntu.com wily/main armhf Packages [1387 kB]
Get:3 http://ports.ubuntu.com wily/restricted armhf Packages [7572 B]
Get:4 http://ports.ubuntu.com wily/multiverse armhf Packages [117 kB]
Get:5 http://ports.ubuntu.com wily/universe armhf Packages [6582 kB]
Get:6 http://ports.ubuntu.com wily/main Translation-en [839 kB]
Get:7 http://ports.ubuntu.com wily/multiverse Translation-en [107 kB]
Get:8 http://ports.ubuntu.com wily/restricted Translation-en [4296 B]
Get:9 http://ports.ubuntu.com wily/universe Translation-en [4579 kB]
Fetched 13.8 MB in 19s (702 kB/s)
Reading package lists... Done

/# echo -e "\nauto eth0\niface eth0 inet dhcp" >>/etc/network/interfaces

/# passwd
(.. some new root password ..)

/# exit
(...from chroot...)
```

**Kernel**

Now what's missing is a kernel to boot from. We can select one from launchpad,
head to `https://launchpad.net/ubuntu/wily/armhf`, search for the term `linux-image`.
We choose a generic-lpae 4.0.2-19 and copy it into the chroot.

```bash
$ wget http://launchpadlibrarian.net/225849170/linux-image-4.2.0-19-generic-lpae_4.2.0-19.23_armhf.deb
$ sudo cp linux-image-4.2.0-19-generic-lpae_4.2.0-19.23_armhf.deb qemu-img/
```

Then going into the chroot (!), installing the kernel there:

```bash
~/vexpress$ sudo chroot qemu-img/
/# dpkg -i linux-image-4.2.0-19-generic-lpae_4.2.0-19.23_armhf.deb
Selecting previously unselected package linux-image-4.2.0-19-generic-lpae.
(Reading database ... 12844 files and directories currently installed.)
Preparing to unpack linux-image-4.2.0-19-generic-lpae_4.2.0-19.23_armhf.deb ...
/usr/bin/locale: Cannot set LC_CTYPE to default locale: No such file or directory
/usr/bin/locale: Cannot set LC_MESSAGES to default locale: No such file or directory
/usr/bin/locale: Cannot set LC_ALL to default locale: No such file or directory
Done.
Unpacking linux-image-4.2.0-19-generic-lpae (4.2.0-19.23) ...
Setting up linux-image-4.2.0-19-generic-lpae (4.2.0-19.23) ...
Running depmod.
update-initramfs: deferring update (hook will be called later)
Examining /etc/kernel/postinst.d.
run-parts: executing /etc/kernel/postinst.d/apt-auto-removal 4.2.0-19-generic-lpae /boot/vmlinuz-4.2.0-19-generic-lpae
run-parts: executing /etc/kernel/postinst.d/initramfs-tools 4.2.0-19-generic-lpae /boot/vmlinuz-4.2.0-19-generic-lpae
update-initramfs: Generating /boot/initrd.img-4.2.0-19-generic-lpae
```

This results in some additional files being present, which we need outside of the chroot,
because they're feeded to the qemu-system-arm emulator:

 * boot/vmlinuz-4.2.0-19-generic-lpae
 * boot/initrd.img-4.2.0-19-generic-lpae
 * lib/firmware/4.2.0-19-generic-lpae/device-tree/vexpress-v2p-ca15-tc1.dtb

Exit chroot, copy these to our `vexpress` base directory. After this we can umount
the `qemu-img` chroot directory.

```bash
~/vexpress$ sudo cp qemu-img/boot/vmlinuz-4.2.0-19-generic-lpae qemu-img/boot/initrd.img-4.2.0-19-generic-lpae qemu-img/lib/firmware/4.2.0-19-generic-lpae/device-tree/vexpress-v2p-ca15-tc1.dtb .
~/vexpress$ sudo umount qemu-img
```

**Booting**

The image is ready by now, so it can be booted. The command is quite long, we're feeding
  * the kernel image, init.rd, the firmware file
  * a machine type (vepxress-a15)
  * memory and serial console settings
  * take the image as a drive (-sd)
  * a port forwarding to ssh into it

into qemu-system-arm:

```bash
~/vexpress$ sudo qemu-system-arm \
  -kernel vmlinuz-4.2.0-19-generic-lpae \
  -initrd initrd.img-4.2.0-19-generic-lpae \
  -dtb ./vexpress-v2p-ca15-tc1.dtb \
  -machine vexpress-a15 -serial stdio -m 1024 \
  -append 'root=/dev/mmcblk0 rw mem=1024M earlyprintk=serial raid=noautodetect rootwait console=ttyAMA0,38400n8 devtmpfs.mount=0' \
  -net nic -net user,hostfwd=tcp::2223-:22 \
  -sd vexpress-4G.img
```

After firiing this off, the system starts to boot. An additional qemu window
pops up, being the terminal console.

```
pulseaudio: set_sink_input_volume() failed
pulseaudio: Reason: Invalid argument
pulseaudio: set_sink_input_mute() failed
pulseaudio: Reason: Invalid argument
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Initializing cgroup subsys cpuacct
[    0.000000] Linux version 4.2.0-19-generic-lpae (buildd@kishi18) (gcc version 5.2.1 20151010 (Ubuntu 5.2.1-22ubuntu2) ) #23-Ubuntu SMP Wed Nov 11 13:56:48 UTC 2015 (Ubuntu 4.2.0-19.23-generic-lpae 4.2.6)
[    0.000000] CPU: ARMv7 Processor [412fc0f1] revision 1 (ARMv7), cr=30c5387d
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, PIPT instruction cache
(...)
[  OK  ] Started Serial Getty on ttyAMA0.
[  OK  ] Started Getty on tty1.
[  OK  ] Reached target Login Prompts.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
[  OK  ] Started Stop ureadahead data collection 45s after completed startup.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Started Update UTMP about System Runlevel Changes.
(...)

login:
```

And we're able to log in using root (or whatever account has been set)!

```bash
~# uname -a
Linux W530 4.2.0-19-generic-lpae #23-Ubuntu SMP Wed Nov 11 13:56:48 UTC 2015 armv7l armv7l armv7l GNU/Linux

~# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 15.10
Release:	15.10
Codename:	wily
```

## Notes

The are other approaches, also described by articles in ubuntu's wiki, by
working with pre-built cloud images. I think this approach here using debootstrap
yields very small OS instance. The sample image above does not include python,
or an ssh server, so it can be made much smaller than the 4GB.

## License

This material is licensed under the Creative Commons Attribution-ShareAlike 3.0 License.
Adapted material from Paolo Pisati (p-pisati on launchpad.net)

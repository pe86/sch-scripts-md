ltsp-image
==========

generate a squashfs image from an image source
----------------------------------------------

Date

:   October 2021

Copyright

:   Copyright 2019-2021 the LTSP team, see AUTHORS

Version

:   LTSP 21.01-1

Manual section

:   8

Manual group

:   LTSP Manual

### SYNOPSIS

**ltsp** \[*ltsp-options*\] **image** \[**-b** *backup*\] \[**-c**
*cleanup*\] \[**-i** *ionice*\] \[**-k** *kernel-initrd*\] \[**-m**
*mksquashfs-params*\] \[**-r** *revert*\] \[*image*\] \...

### DESCRIPTION

Compress a virtual machine image or chroot directory into a squashfs
image, to be used as the network root filesystem of LTSP clients. It\'s
used in similar fashion to live CDs, i.e. all clients will boot from
this single read only image and then use SSHFS or NFS to mount
/home/username from the server.

### OPTIONS

See the **ltsp(8)** man page for *ltsp-options*.

-c, \--cleanup=\<0\|1\>

:   Create a writeable overlay on top of the image source and
    temporarily remove user accounts and sensitive data before calling
    mksquashfs. Defaults to 1.

-b, \--backup=\<0\|1\>

:   Backup /srv/ltsp/images/*image*.img to *image*.img.old. Defaults
    to 1.

-i, \--ionice=cmdline

:   Set a prefix command to run mksquashfs with a lower priority, or
    specify \"\" to disable it completely. Defaults to
    `nice ionice -c3`.

-k, \--kernel-initrd=glob-regex

:   Pass this parameter to the `ltsp kernel` call after the squashfs
    creation. See ltsp-kernel(8) for more information.

-m, \--mksquashfs-params=\<\"params\"\>

:   Pass *\$params* to the mksquashfs call unquoted; so *params*
    shouldn\'t contain spaces. See **mksquashfs(1)** for more
    information.

-r, \--revert\<=\[0\|1\]\>

:   Move /srv/ltsp/images/*image*.img.old to *image*.img and call
    `ltsp kernel image`. Useful when the clients won\'t boot with the
    new image.

### IMAGE TYPES

There are three \"image\" types in LTSP, in the following locations. The
/srv/ltsp path can be configured using `ltsp --base-dir=`:

/srv/ltsp/*img\_name*.img

:   Source images are placed directly under /srv/ltsp and usually are
    symlinks to virtual machine raw disk files. They\'re only used by
    `ltsp image`.

/srv/ltsp/*img\_name*

:   Chroot directories can be used both as sources for `ltsp image` and
    as NFS root exports for the clients.

/srv/ltsp/images/*img\_name*.img

:   Exported images (usually squashfs) are placed under the images
    directory and the clients can netboot from them.

Images can be specified as simple names like `ltsp image img_name`, in
which case the aforementioned locations are searched, or as or full
paths like `ltsp image ~/VMs/vm.img`.

The supported image types result in the following three methods to use
LTSP. You may use either one of the methods or even all of them at the
same time.

### CHROOTLESS

Chrootless LTSP, previously called \"ltsp-pnp\", is the recommended way
to maintain LTSP **if** its restrictions are acceptable. In this mode,
the server operating system itself is exported into a squashfs file and
used for netbooting all the clients. You, the sysadmin, would use the
typical GUI tools to manage the server, like software centers or update
managers. Then whenever necessary, you\'d run:

``` {.sourceCode .bash}
ltsp image /
```

This creates or updates /srv/ltsp/images/x86\_64.img (the arch name
comes from `uname -m`). Then, all the clients should be able to boot
from x86\_64.img and have a desktop environment identical to the server.

The big advantage of the chrootless mode is simplicity: there are no
virtual machines or chroots involved. You\'d maintain the server like
any \"home desktop PC\", and have all clients be exact replicas, which
is as simple as it gets.

The disadvantages are that the clients need to have the same
architecture as the server (e.g. all x86\_64), and that the server
can\'t be a \"full blown server\" with LDAP and Apache and a lot of
other services, without taking care to disable those services on the
clients with the MASK\_SYSTEM\_SERVICES parameter of ltsp.conf. Note
that MASK\_SYSTEM\_SERVICES already includes Apache and MySQL and a few
other popular services that we don\'t want in LTSP clients, so it\'s not
a problem if you install Apache on the LTSP server.

If for some reason you prefer a different name to `uname -m`, you may
create a symlink:

``` {.sourceCode .bash}
ln -s / ~/amd64
```

\...and run `ltsp image ~/amd64` instead.

### VM IMAGES

If the chrootless case doesn\'t fit you, you may use VirtualBox,
virt-manager, KVM, VMWare and similar tools to maintain one or more
template images for the clients. As an example, let\'s suppose you
create a VM in VirtualBox and call it \"debian\". At the disk creation
dialog, select \"VMDK\" type and \"Fixed size\", not \"Dynamically
allocated\". Proceed with installing Debian on it. When you\'re done,
close VirtualBox and symlink the VM disk so that LTSP finds it more
easily:

``` {.sourceCode .bash}
ln -rs ~/VirtualBox\ VMs/debian/debian-flat.vmdk /srv/ltsp/debian.img
```

To export this image to the clients, after the initial creation or after
updates etc, you\'d run:

``` {.sourceCode .bash}
ltsp image debian
```

It\'s also possible to omit the symlink by running:

``` {.sourceCode .bash}
ltsp image ~/VirtualBox\ VMs/debian/debian-flat.vmdk
```

\...but then the image name shown in the iPXE boot menu would be
\"debian-flat\", which isn\'t pretty.

To sum up, you may symlink raw VM disks in /srv/ltsp/img\_name.img, and
`ltsp image img_name` will allow LTSP clients to netboot from them.
Please also see the DIRECT IMAGES section of ltsp-kernel(8) for an
advanced method of allowing clients to netboot directly from a VM or
.iso image without even running `ltsp image`, and the ADVANCED IMAGE
SOURCES section of **ltsp-ipxe(8)** for extreme cases like telling the
LTSP cliens to boot from an .iso image inside a local disk partition!

### CHROOTS

Chroot directories in /srv/ltsp/img\_name are properly supported as
image sources by LTSP, but their creation and maintenance are left to
external tools like debootstrap, lxc etc. I.e. the `ltsp-build-client`
LTSPv5 tool no longer exists. LTSP users are invited to create
appropriate documentation in the [community
wiki](https://github.com/ltsp/ltsp/wiki/chroots). As a small example,
you can use kvm to netboot a chroot and maintain it if you NFS-export
/srv/ltsp/img\_name in rw mode for your server IP, and then run

``` {.sourceCode .bash}
kvm -m 512 -kernel img_name/vmlinuz -initrd img_name/initrd.img \
     -append "rw root=/dev/nfs nfsroot=192.168.0.10:/srv/ltsp/img_name"
```

### EXAMPLES

Use the server installation as a template to generate a client image
(chrootless, previously called ltsp-pnp):

``` {.sourceCode .bash}
ltsp image /
```

Inform `ltsp image` that a chrootless installation uses separate /boot
and /opt partitions:

``` {.sourceCode .bash}
ltsp image /,,/boot,subdir=boot,,/opt,subdir=opt
```

Compress the /srv/ltsp/x86\_64 chroot or the /srv/ltsp/x86\_64.img
virtual machine image, whichever exists of those two, into
/srv/ltsp/images/x86\_64.img, while disabling ionice:

``` {.sourceCode .bash}
ltsp image --ionice="" x86_64
```

Specify an absolute path to a virtual machine image:

``` {.sourceCode .bash}
ltsp image /home/user/VirtualBox\ VMs/x86_32/x86_32-flat.vmdk
```

Revert to the the previous version of the \"chrootless\" image:

``` {.sourceCode .bash}
ltsp image -r /
```

### SEE ALSO

**ltsp(8)**, **ltsp.conf(5)**, **ltsp-dnsmasq(8)**, **ltsp-image(8)**,
**ltsp-info(8)**, **ltsp-initrd(8)**, **ltsp-ipxe(8)**,
**ltsp-kernel(8)**, **ltsp-nfs(8)**

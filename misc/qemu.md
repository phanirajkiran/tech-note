QEMU
====

# Compile and install

```shell
$ cd /root
$ git clone git://git.qemu-project.org/qemu.git
$ cd qemu
$ git submodule update --init dtc
$ git submodule update --init pixman $./configure
$ make && make install
```

# Run kernel image

To discover QEMU commands, execute `ls /usr/bin/qemu-*`.

```shell
$ qemu-system-i386 -m 16M -boot a -fda Image -hda ../osdi.img
```

## Command line options

- Processor
  - `-cpu CPU`: Specify a processor architecture to emulate
    - check `qemu -cpu ?` for supported processors
  - `-cpu host`: Emulate your host processor **(recommended)**
- RAM
  - `-m MEMORY`: Specify the amount of memory in M or G (default: 128 MB)
- Hard drive
  - `-hda IMAGE.img`: Set a virtual hard drive and use the specified image file for it
  - `-fda Image`: Set a floopy drive and use the specified image file for it
- Boot order
  - `-boot a`: Boot the floopy drive
  - `-boot c`: Boot the first virtual hard drive
  - `-boot d`: Boot the first virtual CDROM drive
  - `-boot n`: Boot from virtual network

Reference: [https://wiki.gentoo.org/wiki/QEMU/Options](https://wiki.gentoo.org/wiki/QEMU/Options)

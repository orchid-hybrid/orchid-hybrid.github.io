---
layout: post
title: Mezzano build tips
---

Here are my notes on getting mezzano to bootstrap. These instructions will probably go out of date as the systems changes and improves.

# Mezzano

To build and bootstrap mezzano you need the sources, SBCL, libfixposix and quicklisp.

* git clone https://github.com/froggey/Mezzano
* http://www.sbcl.org/
* https://github.com/sionescu/libfixposix
* http://www.quicklisp.org/

# SBCL

First install SBCL and enable UTF-8 by adding the following command to `~/.sbclrc`:

```
(setf SB-IMPL::*DEFAULT-EXTERNAL-FORMAT* :utf-8)
```

now install quicklisp:

```
(load "~/Downloads/quicklisp.lisp")
(quicklisp-quickstart:install)
(ql:add-to-init-file)
```

also install the libfixposix library which a lisp IO library later will need. To let SBCL find this library I had to use a command like this:

```
export LD_LIBRARY_PATH="/usr/local/lib:$LD_LIBRARY_PATH"
```

Another thing that might be good to add to the sbcl init file is

```
(push #P"/home/gero/Lisp/Mezzano/" asdf:*central-registry*)
(push #P"/home/gero/Lisp/Mezzano/file-server/" asdf:*central-registry*)
```

so that asdf will be able to load mezzano easily.

with SBCL set up you should be able to go into the mezzano sources and try out the file server.

# Fileserver

You can set up the fileserver and test that it works first.

```
(ql:quickload :lispos-file)
(file-server::spawn-file-server)
```

If you look at `fileserver/server.lisp` you can see that the fileserver listens on port 2599. You can test that the fileserver is working by telnetting into it:

```
telnet 127.0.0.1 2599
```

try typing `(:ping)` for example.

Mezzano itself will connect to this file server from inside the VM later on. When it does this it copies files in from the host system and compiles them.

You can check your local ip address with a command like `ip addr`. It should look something like `192.168.1.7` for example.

# First compile

At this point it would be a good idea to do the first stage of compilation.

```
(ql:quickload :lispos)

(with-compilation-unit ()
  (sys.c::set-up-cross-compiler)
  (mapc 'sys.c::load-for-cross-compiler cold-generator::*supervisor-source-files*)
  (mapc 'sys.c::load-for-cross-compiler cold-generator::*source-files*)
  (mapc 'sys.c::load-for-cross-compiler cold-generator::*warm-source-files*))
```

once this works you need to configure `ipl.lisp`.

# Setting up home/ and ipl.lisp

Edit ipl.lisp replacing the ip address with your local one.

Edit `*default-pathname-defaults*` to point to the mezzano source code tree.

Create a new folder somewhere (it's recommended to put it inside the source tree) called home and a fonts folder by running the following commands:

````
mkdir home
mkdir home/fonts
```

(be careful about case sensitivity, you cannot call the folder Fonts/ or it wont work)

Make mezzano.file-system::*home-directory* point to that before setting it up by copying in all the resources listed in https://github.com/froggey/Mezzano/blob/master/BUILD#L26 (remember to get Mandarin_Pair.jpg!)

If you like you can try building the cold image (and booting it to get a basic REPL) before copying all the resources in:

```
(cold-generator::make-image "mezzano" :header-path "tools/disk_header")
```

# Booting the image in a VM

It's easier to quickly test a boot in qemu with the following command:

```
qemu-system-x86_64 -m 512 -hda mezzano.image -serial stdio -net user -net nic,model=virtio -vga std
```

on the other hand, virtual box is much much faster.

Here's a checklist of things that the VM needs:

Mezzano is 64 bit and needs at least 512 MB of memory to work.

You will need to use virtio-net drivers:

* Qemu: the flags `-net user -net nic,model=virtio` do this
* Virtual Box: Serial Ports > Network > Advanced > Adapter Type: Paravirtualized network (virtio-net)

It's really important to be able to read serial port output so that you can understand how the boot process is going.

* Qemu: The flag -serial stdio gives you this.
* Virtual Box: Serial Ports > Enable Serial Port > Port Mode: Raw File + add a path for the file.

You can view this file inside emacs with M-x auto-revert-mode on to get the latest changes while you boot.

To create a virtualbox image from the raw image:

```
qemu-img convert -f raw -O vmdk mezzano.image mezzano.vmdk 
```

or

```
VBoxManage convertfromraw --format vmdk mezzano.image mezzano.vmdk
```

To create a VBox system make a new machine called mezanno with type Other, Version Other/Unknown (64-bit) use the vmdk image.

# Bootstrapping

The bootstrapping is mostly automatic but takes a long time (about two hours) but you will need to watch the file server and serial port for any mistakes in setting things up.

The networking might screw up when you just boot into the system. If so just reboot, or you can try pinging sites in the debug REPL

```
(sys.net::ping-host "google.com")
```

and pinging yourself

```
(sys.net::ping-host "192.168.1.7")
```

once that's working try a reboot.

It is helpful to test connecting to the fileserver from inside a linux VM to make sure virtio is set up correctly.

It is expected to error once with undefined-function EXPORT, there is a note in ipl.lisp about it. just reboot.

Once you get in you can use alt-drag to move windows around.

# RCU Binding for Lunatik
## Overview:

Lunatik is a programming environment based on the Lua language for kernel scripting. By allowing kernel scripting, a user may create scripts in Lua to customize their kernel to suit their needs. Modern day kernels, such as Linux, work on a highly concurrent environment, and therefore must use robust synchronization APIs to ensure data consistency, with each API having their own use cases.

The Read-copy-update (RCU) is one of those APIs, made specifically for scenarios where data reading is much more common than writing. RCU allows concurrent readers to access protected data in a non-blocking way even during updates or removals. RCU, as with the rest of the Linux Kernel, is written in C.

This project creates a Lua binding to RCU for use in Lunatik, allowing Lua data to be safely shared, accessed and modified among concurrent Lua states.

## Installation:

Lunatik is presented as an in-tree kernel module, meaning we'll have to compile our own custom kernel to use it. This source tree contains the Lunatik source, the poc-driver and the RCU Binding. The poc-driver is the driver user to load Lua scripts from user space to kernel space and resides in lunatik/poc-driver. The RCU binding resides in the lunatik/rcu directory.

During the compilation of Lunatik, both the driver and the binding are also compiled and installed. 

This instructions are based on the Debian distro and it is the environment I've used during the GSOC 2018 period.

First, you'll need to install the linux headers for your running kernel. These files are needed to develop kernel modules. To see which kernel you're running, type:
```bash
$ uname -r
```

After that, install the appropriate headers package. The package names may change from each distro.
```bash
$ sudo apt install linux-headers-$(uname -r)
```

Also install:
```bash
$ sudo apt install build-essential libncurses-dev linux-source
```

libncurses is a package that allows us to us a GUI when configuring the kernel, and linux-source is a debian package that contains the kernel source. This package creates a linux-source-xx.tar.xz (where xx is your running kernel version) in /usr/src. Extract this file.
```bash
$ sudo tar -xf linux-source-xx.tar.xz
```

You should now have a directory /usr/src/linux-source-xx.

Now, we have to add the Lunatik files to the kernel source. Download the Lunatik source files (https://github.com/caioluiz/lunatik) and copy them to the linux source drivers directory, located in /usr/src/linux-source-xx/drivers.

Edit the Kconfig file located in the drivers directory and add "source lunatik/Kconfig" at the end of it.

In that same directory, also edit the makefile and add "obj-$(CONFIG_LUNATIK) += lunatik/" at the end.

Go back to /usr/src/linux-source-xx and run 
```bash
$ sudo make menuconfig
```
A gui will appear with various kernel configurations. Go to device drivers options and at the end of that list, find Lunatik and enable it with module support. This will also enable the poc-driver automatically.
Also go to "Processor type and features", then "Preemption Model" and choose preemptible Kernel.
A preemptible kernel will allow for multiple Lua scripts to execute and interact with the rcu hash table concurrently.

With Lunatik now enabled, we now have to compile the entire kernel. Fortunately, debian offers us a simple and quick approach.
```bash
$ make -j4 bindeb-pkg ARCH=x86_64
```
This command will create a .deb file that can be normally installed like any other package.
-j flag sets the number of cores used during the compilation, notice that this is a process that can take quite some time.

The result will be a file named linux-image-xx.deb. Install it as a normal package and reboot the system.
```bash
$ dpkg -i linux-image-xx.deb
```

Now Lunatik can be used as a normal module. To install it, use 
```bash 
$ sudo modprobe -v lunatik
``` 
and to remove it:
```bash
$ sudo modprobe -r lunatik
```

To see if it's currently running, type:
```bash
$ lsmod | grep lunatik
```

Since it's a kernel module, Lunatik will not print messages to a normal terminal. To see them, use either dmesg or journalctl to have access to the kernel and driver messages.
```bash
$ sudo dmesg
$ sudo journalctl -k
```

When Lunatik is loaded, the poc-driver is also loaded, called luadrv in the filesystem and is located in /dev/luadrv. This driver expects Lua scripts to execute them in kernel space. If your file is in user space, you can redirect it using
```bash
$ cat script.lua > /dev/luadrv
```
and the file will copied to kernel space and then executed.

If you modify either of the RCU binding or Lunatik itself, you'll need to recompile the module again. This step will be much quicker now than the first time because only the modified files will be compiled again. To do it, use:
```bash
sudo make modules -j4 ARCH=x86_64
sudo make modules_install
```

## The RCU Binding:
The RCU binding developed in this project defines a hash table that can be accessed concurrently. The elements of this table are indexed by a unique string key and can hold a value of either an int, a bool or a string. These are the values types that can be shared among the Lua states.

The hash table is defined using the kernel own macros. This results in an array where each element is the head of a linked list (a bucket). We can change the size of this array of linked lists at compile time, adding more buckets and reducing collisions if needed. Each of these buckets is protected by RCU independently, meaning that each bucket has it's own lock, and elements from different buckets can be modified at the same time. RCU allows any number of readers and up to one writer in the same bucket.

The binding is exported to Lua via a table called "rcu" that implements the __index and __newindex metamethods. This way, the rcu table can be accessed and modified like a normal Lua table.

The binding also exports a "for_each" function, that applies a function to each element of hash table, without modifying it. RCU allows this operation to be made concurrently with readers. For example, to print all elements present on the table, you can write:
```lua
rcu.for_each(print)
```

Whenever a element is to be accessed, the __index function will call the internal C function rcu_search_element, and whenever a element is to be added or modified, __newindex will call the appropriate internal functions to add, delete or replace the element.

For example, to add an element to the rcu protected hash table from Lua, just write
```lua
rcu["somekey"] = some_value
```
Since the keys are strings, we can also use the usual Lua dot notation 'table.key' to access an element.

To update:
```lua
rcu.somekey = another_value
```

And to delete:
```lua
rcu.somekey = nil
```

To access an element, you can simply use
```lua
print(rcu.somekey)
```

Notice that, in Lua code, you don't need to take care of locks and mutexes manually, as these are handled in the C side and that by utilizing RCU we can guarantee that the data will always be accessible and never corrupted.

# How to write a Linux Syscall in C

## Setup

1. Install and use Virtual Box (you can skip this step if you don't care about breaking your Kernel)

2. Download the Linux distribution of your choice. I used the one from osboxes : [Linux Distributions](https://www.osboxes.org/arch-linux/#archlinux-202109-vbox)

Osboxes provide a lot of pre-installed Linux distributions.

3. Create a new VM in VirtualBox and attach the downloaded ISO file to it.

![VirtualBox Image](virtualBox/1.png)

![VirtualBox Image](virtualBox/2.png)

3. Start the VM and login with the username and password provided in the download page.

> *Note* :
>
> Mine was `osboxes` as username and `osboxes.org` as password.

4. The first preparation step you should take is to install bc, a build-time dependency of Linux that isn’t included in the virtual machine then reboot !

## Download the Linux source code

Download it using curl :
    Don't install a random kernel version, you should choose the correct one depend on your current version.

## Write the system call :

1. Configure the kernel :

The Linux Kernel is extraordinarily configurable; you can enable and disable many of its features, as well as set build parameters. If you were to make every configuration choice manually, you’d be doing it all day. Instead, you can skip this step by simply copying your kernel’s existing configuration.

To ensure that you have values for all configuration variables, run make oldconfig. More than likely, this will not ask you any configuration questions.

```bash
$ make oldconfig
```

The only configuration item that you ought to modify is the kernel name, to ensure that it doesn’t conflict with your currently installed one. On Arch Linux, the kernel is built with the suffix `-ARCH`. You should change this suffix to something unique to you: I used `-MyARCH`. To do this, the simplest way is to open .config with a text editor, and modify this line directly. You’ll find it just under the “General setup” heading, not too far down the file:

```bash
CONFIG_LOCALVERSION="-MyARCH"
```

2. Add the system call in the system call table :

You need to find the file containing the system call table for x86_64. This table is read by scripts and used to generate some of the boilerplate code, which makes our lives a lot easier! Find the correct location to put your system call and by respecting the format (DON'T USE SPACES, ONLY TABULATIONS) write it in the file and save it.

> *Note* :
>
> Each value is separated by tabulations NO SPACES !.

3. Code the system call :

The last step is to write the function for the system call! We haven’t really gone into what the system call should do, but really all we would like is to do something simple that we can observe. An easy thing to do is write to the kernel log using printk(). So, our system call will take one argument, a string, and it will write it to the kernel log.

You can implement system calls anywhere, but miscellaneous syscalls tend to go in the kernel/sys.c file. Put this somewhere in the file:

I would like you to write the syscall by yourself, take exemple of the others syscalls in the file and try to understand how it works.

SYSCALL_DEFINE\<N\> is a family of macros that make it easy to define a system call with N arguments. The first argument to the macro is the name of the system call (without sys_ prepended to it). The remaining arguments are pairs of type and name for the parameters. Since our system call has one argument, we use SYSCALL_DEFINE1, and our only parameter is a char * which we name msg.

An interesting issue that we encounter immediately is that we cannot directly use the msg pointer provided to us. There are several reasons why this is the case, but none are very obvious!

- The process could try to trick us into printing out data from kernel memory by giving us a pointer that maps to kernel space. This should not be allowed.
- The process could try to read another process’s memory by giving a pointer that maps into another process’s address space.
- We also need to respect the read/write/execute permissions of memory.

To handle these issues, we use a handy `strncpy_from_user()` function which behaves like normal strncpy, but checks the user-space memory address first. If the string was too long or if there was a problem copying, we return EFAULT (although returning EINVAL for a too-long string might be better).

Finally, we use printk with the KERN_INFO log level. This is actually a macro that resolves to a string literal. The compiler concatenates that with the format string and printk() uses it to determine the log level. Finally, printk does formatting similar to printf(), which is where the %s comes in.

4. Build and deploy the kernel :

This part is a bit more complicated so I write a script to do it for you. 


deploy.sh
```bash
#!/usr/bin/bash
# Compile and "deploy" a new custom kernel from source on Arch Linux

# Change this if you'd like. It has no relation
# to the suffix set in the kernel config.
SUFFIX="-MyARCH"

# This causes the script to exit if an error occurs
set -e

# Compile the kernel
make
# Compile and install modules
make modules_install

# Install kernel image
cp arch/x86_64/boot/bzImage /boot/vmlinuz-linux$SUFFIX

# Create preset and build initramfs
sed s/linux/linux$SUFFIX/g \
    </etc/mkinitcpio.d/linux.preset \
    >/etc/mkinitcpio.d/linux$SUFFIX.preset
mkinitcpio -p linux$SUFFIX

# Update bootloader entries with new kernels.
grub-mkconfig -o /boot/grub/grub.cfg
```

You can use it with the following command :

```bash
$ chmod u+x deploy.sh
$ sudo ./deploy.sh
```

Once the script is complete run `reboot`

#Custom fedora install

In this post I’ll cover custom fedora linux installation using livecd and kickstart file. I think fedora is one of the best linux distributions around. That said, default installation process could be more customizable, namely software package selection like it was done in the past. Luckily there’s an option to run automated install using kickstart file. 

I boot machine using fedora live iso image I’ve downloaded earlier. During the install I’ll only use terminal. To the kernel boot line in grub boot screen I add ``systemd.unit=multi-user.target``. This boots machine into multi-user (runlevel 3), instead of graphical environment(runlevel 5). Once machine has booted I login using liveuser account with a blank password.
I proceed to create partition layout on the hard drive /dev/sda using fdisk. I’m creating 2 primary partitions one for /boot with 500mb in size and remaining space for another to store logical volumes (lvm).

# Prepare storage
    [liveuser@localhost ~]$ sudo fdisk /dev/sda
    [sudo] password for liveuser:

    Welcome to fdisk (util-linux 2.25.2).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.


    Command (m for help): p
    Disk /dev/sda: 8 GiB, 8589934592 bytes, 16777216 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0xb8b0be00



    Command (m for help): n
    Partition type
       p   primary (0 primary, 0 extended, 4 free)
       e   extended (container for logical partitions)
    Select (default p):

    Using default response p.
    Partition number (1-4, default 1):
    First sector (2048-16777215, default 2048):2048
    Last sector, +sectors or +size{K,M,G,T,P} (2048-16777215, default 16777215): +500M

    Created a new partition 1 of type 'Linux' and of size 500 MiB.

    Command (m for help): n
    Partition type
       p   primary (1 primary, 0 extended, 3 free)
       e   extended (container for logical partitions)
    Select (default p):

    Using default response p.
    Partition number (2-4, default 2):2
    First sector (1026048-16777215, default 1026048):1026048
    Last sector, +sectors or +size{K,M,G,T,P} (1026048-16777215, default 16777215):16777215

    Created a new partition 2 of type 'Linux' and of size 7.5 GiB.

    Command (m for help): write
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Syncing disks.
    
Partition layout is created, next step is creating file systems. I format /dev/sda1 partition in ext4, it will be used as /boot

    [liveuser@localhost ~]$ sudo mkfs.ext4 /dev/sda1
    mke2fs 1.42.11 (09-Jul-2014)
    Creating filesystem with 1968896 4k blocks and 492880 inodes
    Filesystem UUID: 8277a9aa-1c16-4872-984e-f4192b3e56ac
    Superblock backups stored on blocks:
            32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

    Allocating group tables: done
    Writing inode tables: done
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done

Fedora comes with lvm support by default, I let lvm use /dev/sda2. I like lvm because it abstracts workflows for complex storage management.

* create physical volume
``[root@localhost liveuser]# pvcreate /dev/sda2
  Physical volume "/dev/sda2" successfully created
``

* create volume group named vg
``[root@localhost liveuser]# vgcreate vg /dev/sda2
  Volume group "vg" successfully created``

* Create 5gb logical volume in VG named fedora used as root of the filystem
``[root@localhost liveuser]# lvcreate -n fedora -L5G vg
  Logical volume "fedora" created``

* Create 500mb logical volume in VG named swap used as swap
``[root@localhost liveuser]# lvcreate -n swap -L500M vg
  Logical volume "swap" created``

Use lvs command to see resulting lvm layout
``[root@localhost liveuser]# lvs -a
  LV     VG   Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  fedora vg   -wi-a-----   5.00g
  swap   vg   -wi-ao---- 500.00m``

#Kickstart file

    #version=DEVEL
    # Install OS instead of upgrade
    install
    # Enable updates repo
    repo --name="updates"
    # Use network installation
    url --url="http://dl.fedoraproject.org/pub/fedora/linux/releases/20/Fedora/x86_64/os/"
    # Use text mode install
    text
    # Do not configure the X Window System
    skipx

    # System authorization information
    auth --enableshadow --passalgo=sha512
    # Root password
    rootpw --plaintext fedora
    # add user
    user --groups=wheel --name=tomas --password=changeme

    # Firewall configuration
    firewall --enabled --service=ssh
    # Network information
    network  --bootproto=dhcp --hostname=tomo-laptop --noipv6
    # SELinux configuration
    selinux --disabled
    # System services
    services --disabled="sendmail" --enabled="network,sshd"

    # Keyboard layouts
    keyboard --vckeymap=us --xlayouts='us'
    # System language
    lang en_US.UTF-8
    # System timezone
    timezone Australia/Brisbane --isUtc

    # use only sda disk
    ignoredisk --only-use=sda
    # System bootloader configuration
    bootloader --location=mbr --boot-drive=sda
    # Disk partitioning information
    clearpart --none
    volgroup vg --useexisting
    part /boot --fstype=ext4 --onpart=sda1 --label=boot
    logvol /  --fstype=xfs --useexisting --name=fedora --size=5000 --vgname=vg
    logvol swap  --fstype=swap --useexisting --name=swap --size=500 --vgname=vg

    %packages
    @core
    vim
    %end

    %post --logfile /tmp/anaconda_post.log
    # exec post install script
    curl -k https://dl.dropboxusercontent.com/u/32633398/post.py | python -
    %end

    # disable system setup on firstboot
    firstboot --disable
    # Halt after installation
    halt
# Installation process
Anaconda is fedora installer, by default it runs in graphical interface, but it’s possible to run it in text mode and specify kickstart (answer) file to be used for automated installation.

```[root@localhost liveuser]# anaconda -T --kickstart=fedora.ks
Starting installer, one moment...
anaconda 21.48.21-1 for anaconda bluesky (pre-release) started.
 * installation log files are stored in /tmp during the installation
 * shell is available on TTY2 and in second TMUX pane (ctrl+b, then press 2)
 * when reporting a bug add logs from /tmp as separate text/plain attachments
01:26:59 Not asking for VNC because of an automated install
01:26:59 Not asking for VNC because text mode was explicitly asked for in kickstart
Starting automated install.....................................................................
Generating updated storage configuration
Checking storage configuration...
================================================================================
================================================================================
Installation

 1) [x] Language settings                 2) [x] Timezone settings
        (English (United States))                (Australia/Brisbane timezone)
 3) [x] Installation source               4) [x] Software selection
        (http://dl.fedoraproject.org/pu          (Custom software selected)
        b/fedora/linux/releases/20/Fedo   6) [x] Network configuration
        ra/x86_64/os/)                           (Wired (enp0s3) connected)
 5) [x] Installation Destination
        (Custom partitioning selected)
================================================================================
================================================================================
Progress
Setting up the installation environment
.
Creating swap on /dev/mapper/vg-swap
.
Creating xfs on /dev/mapper/vg-fedora
.
Creating ext4 on /dev/sda1
.
Starting package installation process
```

# End

Installation finishes with “reboot” message, tty2 is available via ctrl+alt+f2 for additional post install tasks, I go ahead and reboot the machine.

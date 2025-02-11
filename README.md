I do not take credit for writing this. The majority of this guide is pulled from the Arch Wiki; the rest is from man pages and random reddit/Arch forum threads. I compiled this for myself to have a repeatable process while skipping the weeks of reading on troubleshooting and best practices. This isn't perfect and civil discussions are welcome.

# ZFS on Arch Install Guide

There are some assumptions being made in this guide.
    - I made my own ZFS archiso which is why I copy pacman mirror files from the iso post pacstrap.
    - I chose to run the DKMS modules.
    - I did not remove all my personal decisions like using nano over vi. Make your own educated adjustments.
    - I do not install the bootloader on ZFS. That was an added complexity I didn't think was worth the effort.

**Disable Secure Boot**

https://wiki.archlinux.org/title/Installation_guide

**Connect to the internet**

To set up a network connection in the live environment, go through the following steps:

   Ensure your network interface is listed and enabled, for example with ip-link(8):
````
    ip link

````
   The connection may be verified with ping:
````
    ping archlinux.org

````
Note: In the installation image, systemd-networkd, systemd-resolved, iwd and ModemManager are preconfigured and enabled by default. That will not be the case for the installed system.

**Update the system clock**

In the live environment systemd-timesyncd is enabled by default and time will be synced automatically once a connection to the internet is established.

Use timedatectl(1) to ensure the system clock is synchronized:
````
    timedatectl

````

**Initialize pacman**
```
pacman-key --init
```
```
pacman -Syyuu
```

**Partition the disks**

When recognized by the live system, disks are assigned to a block device such as /dev/sda, /dev/nvme0n1 or /dev/mmcblk0. To identify these devices, use lsblk or fdisk.
````
    fdisk -l

````
Check sector size
https://wiki.archlinux.org/title/Advanced_Format
````
    lsblk -td
````
Assumption is 4096 bytes

Extra NVMe check
```
    nvme id-ns -H /dev/nvme0n1 | grep "Relative Performance"
```
*Metadata Size is the number of extra metadata bytes per logical block address (LBA). As this is not well supported under Linux, it is best to select a format with a value of 0 here.*

*Relative Performance indicates which format will provide degraded, good, better or best performance.*

To change the logical block address size, use nvme format and specify the preferred value with the --lbaf parameter:
```
nvme format --lbaf=X /dev/nvme0n1
```

Results ending in rom, loop or airootfs may be ignored. mmcblk* devices ending in rpbm, boot0 and boot1 can be ignored.
Note: If the disk does not show up, make sure the disk controller is not in RAID mode.
Tip: Check that your NVMe drives and Advanced Format hard disk drives are using the optimal logical sector size before partitioning.

The following partitions are required for a chosen device:

    One partition for the root directory /.
    For booting in UEFI mode: an EFI system partition.

Use a partitioning tool like fdisk to modify partition tables. For example:
```
    fdisk /dev/the_disk_to_be_partitioned

```
*To use size in fdisk the Last sector argument format is "+10G"*

**Note:**

    Take time to plan a long-term partitioning scheme to avoid risky and complicated conversion or re-partitioning procedures in the future.
    If you want to create any stacked block devices for LVM, system encryption or RAID, do it now.
    If the disk from which you want to boot already has an EFI system partition, do not create another one, but use the existing partition instead.
    Swap space can be set on a swap file for file systems supporting it. Alternatively, disk based swap can be avoided entirely by setting up swap on zram after installing the system.

**Example layouts:**
UEFI with GPT Mount point on the installed system 	
| Partition | Partition type | Suggested size |
| ----------|----------------|----------------|
| /boot 	| /dev/efi_system_partition | EFI system partition 	1 GiB |
| SWAP  	| /dev/swap_partition | Linux swap 	At least 4 GiB |
| / 	    | /dev/root_partition | Linux x86-64 root (/) Remainder of the device. At least 23–32 GiB. |

    Other mount points, such as /efi, are possible, provided that the used boot loader is capable of loading the kernel and initramfs images from the root volume. See the warning in Arch boot process#Boot loader.

BIOS with MBR Mount point on the installed system
| Partition | Partition type | Suggested size |
| ----------|----------------|----------------|
| SWAP  	| /dev/swap_partition | Linux swap 	At least 4 GiB |
| / 	    | /dev/root_partition | Linux x86-64 root (/) Remainder of the device. At least 23–32 GiB. |

See also Partitioning#Example layouts.

**Format the partitions**

If you created an EFI system partition, format it to FAT32 using mkfs.fat(8).
Warning: Only format the EFI system partition if you created it during the partitioning step. If there already was an EFI system partition on disk beforehand, reformatting it can destroy the boot loaders of other installed operating systems.
```
mkfs.fat -F 32 /dev/efi_system_partition

```

**Enable ZED on live CD**

Since this guide assumes the usage of zfs-mount-generator(8), we need to generate the zfs-list cache before first booting into our system. This requires:

    Enabling zfs-zed.service on live CD.
```
systemctl enable zfs-zed.service
swapoff --all
```

**Create the root pool**

See ZFS#Creating ZFS pools for detailed info. As an example, the following command creates a root pool named rpool on the partition /dev/nvme0n1p2 and sets the altroot property to /mnt:

# Save the headache and potential issues and do NOT encrypt the pool

This is what I did, but make your own choices:
```
zpool create 
    -o ashift=12 
    -o autotrim=on 
    -O dnodesize=auto
    -O acltype=posixacl    
    -O relatime=on               
    -O normalization=formD 
    -O recordsize=1M       
    -O xattr=sa
    -O compression=zstd             
    -R /mnt
    (rpool name) /dev/root_partition

```

**ashift is IMMUTABLE**
The ashift values range from 9 to 16, with a default value of 0 which *should* auto-detect sector size. 
*The use of ashift=12 is recommended here because many drives today have 4 KiB (or larger) physical sectors, even though they present 512 B logical sectors. Also, a future replacement drive may have 4 KiB physical sectors (in which case ashift=12 is desirable) or 4 KiB logical sectors (in which case ashift=12 is required). In case of 8k sectors or if seeing performance issues use ashift=13*

Setting -O acltype=posixacl enables POSIX ACLs globally. If you do not want this, remove that option, but later add -o acltype=posixacl (note: lowercase “o”) to the zfs create for /var/log, as journald requires ACLs Also, disabling ACLs apparently breaks umask handling with NFSv4.

Setting relatime=on is a middle ground between classic POSIX atime behavior (with its significant performance impact) and atime=off (which provides the best performance by completely disabling atime updates). Since Linux 2.6.30, relatime has been the default for other filesystems.

recordsize default is 128Kb recommended to be 1M for laptop usecase 
    - https://www.reddit.com/r/zfs/comments/gzbb06/dataset_recordsize_for_game_library/
    - https://jrs-s.net/2019/04/03/on-zfs-recordsize/

Setting normalization=formD eliminates some corner cases relating to UTF-8 filename normalization. It also implies utf8only=on, which means that only UTF-8 filenames are allowed. If you care to support non-UTF-8 filenames, do not use this option. For a discussion of why requiring UTF-8 filenames may be a bad idea, see 
    - The problems with enforced UTF-8 only filenames: https://utcc.utoronto.ca/~cks/space/blog/linux/ForcedUTF8Filenames

Setting xattr=sa vastly improves the performance of extended attributes. Inside ZFS, extended attributes are used to implement POSIX ACLs. Extended attributes can also be used by user-space applications. They are used by some desktop GUI applications. They can be used by Samba to store Windows ACLs and DOS attributes; they are required for a Samba Active Directory domain controller. Note that xattr=sa is Linux-specific. If you move your xattr=sa pool to another OpenZFS implementation besides ZFS-on-Linux, extended attributes will not be readable (though your data will be). If portability of extended attributes is important to you, omit the -O xattr=sa above. Even if you do not want xattr=sa for the whole pool, it is probably fine to use it for /var/log.

zstd early abort uses lz4 on first pass since openZFS 2.2

Tip: Use the altroot property, which is set via the -R flag during pool creation or import to temporarily add a prefix to the mount points to avoid shadowing the live cd environment.

**Create filesystems**

See ZFS#Creating datasets for detailed info. Here are some considerations when choosing your dataset options and layouts:

    Most properties are inherited from parent dataset by child unless explicitly overridden.
    The default value of the mountpoint property of the child is <mountpoint of parent>/<name of child>

**Creating datasets**

Users can optionally create a dataset under the zpool as opposed to manually creating directories under the zpool. Datasets allow for an increased level of control (quotas for example) in addition to snapshots. To be able to create and mount a dataset, a directory of the same name must not pre-exist in the zpool. To create a dataset, use:

Tip: Use the altroot property, which is set via the -R flag during pool creation or import to temporarily add a prefix to the mount points to avoid shadowing the live cd environment.

The attributes should be inhereted from pool but it doesn't hurt to call them explicitly
(Again this is what I did, make your own choices):
```
zfs create 
    -o acltype=posixacl    
    -o relatime=on         
    -o dnodesize=auto      
    -o normalization=formD 
    -o recordsize=1M       
    -o xattr=sa            
    -o compression=zstd    
    -o encryption=on -o keyformat=passphrase 
    -o mountpoint=/
    (rpool name)/(dataset name)

```
**Passphrase can be changed later, it's just easier for initial booting**
(my later plan is to migrate from passphrase to key but this is easier done post install)
It is then possible to apply ZFS specific attributes to the dataset. (attributes I want initially are declared above) For example, one could assign a quota limit to a specific directory within a dataset:
```
zfs set quota=20G <nameofzpool>/<nameofdataset>/<directory>
```
(quota is useful if doing a zfs /tmp or /var)
To see all the commands available in ZFS, see zfs(8) or zpool(8). 

**Mount the file systems**

Show mountpoints to adjust as necessary:
```
zfs list
```
Mount the root volume to /mnt. For example, if the root volume is /dev/root_partition:~~
```
zfs set mountpoint=/mnt (pool)/(dataset)

mount --mkdir /dev/boot_partition /mnt/boot

```
Create any remaining mount points under /mnt (such as /mnt/boot for /boot) and mount the volumes in their corresponding hierarchical order.
Tip: Run mount(8) with the --mkdir option to create the specified mount point. Alternatively, create it using mkdir(1) beforehand.

# Install and configure Arch Linux

**Follow the steps of installation guide from Installation guide#Installation to before reboot.**

**Select the mirrors**

Packages to be installed must be downloaded from mirror servers, which are defined in /etc/pacman.d/mirrorlist. On the live system, after connecting to the internet, reflector updates the mirror list by choosing 20 most recently synchronized HTTPS mirrors and sorting them by download rate.

The higher a mirror is placed in the list, the more priority it is given when downloading a package. You may want to inspect the file to see if it is satisfactory. If it is not, edit the file accordingly, and move the geographically closest mirrors to the top of the list, although other criteria should be taken into account.

This file will later be copied to the new system by pacstrap, so it is worth getting right.

Install essential packages
Note: No software or configuration (except for /etc/pacman.d/mirrorlist) gets carried over from the live environment to the installed system.
Use the pacstrap(8) script to install the base package, Linux kernel and firmware for common hardware:
```
pacstrap -K /mnt amd-ucode base base-devel linux-lts linux-lts-headers linux-firmware networkmanager lsusb usbutils nano archzfs-dkms zfs-dkms zfs-utils

```
Tip:
You can substitute linux with a kernel package of your choice, or you could omit it entirely when installing in a container.
You could omit the installation of the firmware package when installing in a virtual machine or container.
The base package does not include all tools from the live installation, so installing more packages may be necessary for a fully functional base system. To install other packages or package groups, append the names to the pacstrap command above (space separated) or use pacman to install them while chrooted into the new system. In particular, consider installing:

- CPU microcode updates, amd—ucode or intel-ucode, for hardware bug and security fixes,
    - amd-ucode for laptop
- userspace utilities for file systems that will be used on the system—for the purposes of e.g. file system creation and fsck,
utilities for accessing and managing RAID or LVM if they will be used on the system,
    - covered by zfs-utils
- specific firmware for other devices not included in linux-firmware (e.g. sof—firmware for onboard audio, linux-firmware-marvell for Marvell wireless and any of the multiple firmware packages for Broadcom wireless),
    - skipping dedicated wireless and audio at this stage
- software necessary for networking (e.g. a network manager or a standalone DHCP client, authentication software for Wi-Fi, ModemManager for mobile broadband connections),
    - networkmanager
- a console text editor (e.g nano) to allow editing configuration files from the console,
    - nano
- packages for accessing documentation in man and info pages: man-db, man-pages and texinfo.
    - good idea but skipping due to phone/other computer availability
For comparison, packages available in the live system can be found in pkglist.x86_64.txt.

**Copy files from iso to /mnt**
```
cp /etc/pacman.d/archzfs_mirrorlist /mnt/etc/pacman.d/
cp /etc/pacman.conf /mnt/etc/pacman.conf
rsync -a /usr/share/pacman/keyrings/ /mnt/usr/share/pacman/keyrings
```

**Fstab**

Generate an fstab file (use -U or -L to define by UUID or labels, respectively):
```
genfstab -U /mnt >> /mnt/etc/fstab
```
Check the resulting /mnt/etc/fstab file, and edit it in case of errors.

**Chroot**

Change root into the new system:
```
arch-chroot /mnt
```

**Import archzfs keys and sync databases**
```
pacman-key --init
pacman-key --populate && pacman -Syu
```

**Time**

Set the time zone:
```
ln -sf /usr/share/zoneinfo/america/boise /etc/localtime
```
Check hardware clock is UTC:
```
hwclock --show
```
if not:
``
timedatectl set-local-rtc 0
``
Run hwclock(8) to generate /etc/adjtime:
```
hwclock --systohc
hwclock --adjtime
```
This command assumes the hardware clock is set to UTC. See System time#Time standard for details.

To prevent clock drift and ensure accurate time, set up time synchronization using a Network Time Protocol (NTP) client such as systemd-timesyncd.

```
timedatectl set-timezone America/Boise
```
Verify **RTC in Local TZ: NO**
```
timedatectl
```
**Localization**

**Edit /etc/locale.gen**
uncomment en_US.UTF-8 UTF-8 and other needed UTF-8 locales. Generate the locales by running:
```
locale-gen
```
Create the locale.conf(5) file, and set the LANG variable accordingly:
   /etc/locale.conf
```
localectl set-locale LANG=en_US.UTF-8
```
If you set the console keyboard layout, make the changes persistent in vconsole.conf(5):
/etc/vconsole.conf

    KEYMAP=us

**Network configuration**

Create the hostname file:
/etc/hostname

    yourhostname

Complete the network configuration for the newly installed environment. That may include installing suitable network management software, configuring it if necessary and enabling its systemd unit so that it starts at boot.

**Initramfs**

Creating a new initramfs is usually not required, because mkinitcpio was run on installation of the kernel package with pacstrap.

For LVM, system encryption or RAID, modify mkinitcpio.conf(5) and recreate the initramfs image: **(for zfs we do this later)**
```
mkinitcpio -P
```
**Root password**

Set the root password:
```
passwd
```

**Have boot messages stay on tty1**

By default, Arch has the getty@tty1 service enabled. The service file already passes --noclear, which stops agetty from clearing the screen. However systemd clears the screen before starting it. To disable this behavior, create a drop-in file:

/etc/systemd/system/getty@tty1.service.d/noclear.conf
```
[Service]
TTYVTDisallocate=no
```
**User Management**
```
useradd -m -G wheel <username>
```

**Using visudo**

The configuration file for sudo is /etc/sudoers. It should always be edited with the visudo(8) command. visudo locks the sudoers file, saves edits to a temporary file, and checks it for syntax errors before copying it to /etc/sudoers.
Warning:

    It is imperative that sudoers be free of syntax errors! Any error makes sudo unusable. Always edit it with visudo to prevent errors.
    visudo(8) warns that configuring visudo to honor the user environment variables for their editor of choice may be a security hole, since it allows the user with visudo privileges to run arbitrary commands as root without logging simply by setting that variable to something else.

The default editor for visudo is vi. The sudo package is compiled with --with-env-editor and honors the use of the SUDO_EDITOR, VISUAL and EDITOR variables. EDITOR is not used when VISUAL is set.

To establish nano as the visudo editor for the duration of the current shell session, export EDITOR=nano; to use a different editor just once simply set the variable before calling visudo:
```
EDITOR=nano visudo
```
Alternatively you may edit a copy of the /etc/sudoers file and check it using visudo -c /copy/of/sudoers. This might come in handy in case you want to circumvent locking the file with visudo.

To change the editor of choice permanently system-wide only for visudo, add the following to /etc/sudoers (assuming nano is your preferred editor):
```
# Set default EDITOR to restricted version of nano, and do not allow visudo to use EDITOR/VISUAL.
Defaults      editor=/usr/bin/rnano, !env_editor
```
Add new user to sudoer:
```
<USER_NAME>   ALL=(ALL:ALL) ALL
```

**Boot loader**

Choose and install a Linux-capable boot loader.
    
    - Grub

First, install the packages grub and efibootmgr: GRUB is the boot loader while efibootmgr is used by the GRUB installation script to write boot entries to NVRAM.
```
pacman -Syu grub efibootmgr
```
Then follow the below steps to install GRUB to your disk:

Mount the EFI system partition and in the remainder of this section, substitute esp with its mount point. (should already be mounted)
    - /boot

Choose a boot loader identifier, here named GRUB. A directory of that name will be created in esp/EFI/ to store the EFI binary and this is the name that will appear in the UEFI boot menu to identify the GRUB boot entry.

Execute the following command to install the GRUB EFI application grubx64.efi to esp/EFI/GRUB/ and install its modules to /boot/grub/x86_64-efi/.

Note:
Make sure to install the packages and run the grub-install command from the system in which GRUB will be installed as the boot loader. That means if you are booting from the live installation environment, you need to be inside the chroot when running grub-install. If for some reason it is necessary to run grub-install from outside of the installed system, append the --boot-directory= option with the path to the mounted /boot directory, e.g --boot-directory=/mnt/boot.
Some motherboards cannot handle bootloader-id with spaces in it.
```
grub-install --target=x86_64-efi --efi-directory=boot --bootloader-id=GRUB
```
After the above installation completed, the main GRUB directory is located at /boot/grub/.

**Generate the main configuration file**

This section only covers editing the /etc/default/grub configuration file. See /Tips and tricks for more information.

Note: Remember to always re-generate the main configuration file after making changes to /etc/default/grub and/or files in /etc/grub.d/.

After the installation, the main configuration file /boot/grub/grub.cfg needs to be generated. The generation process can be influenced by a variety of options in /etc/default/grub and scripts in /etc/grub.d/. For the list of options in /etc/default/grub and a concise description of each refer to GNU's documentation.

If you have not done additional configuration, the automatic generation will determine the root filesystem of the system to boot for the configuration file. For that to succeed it is important that the system is either booted or chrooted into.

Note:
The default file path is /boot/grub/grub.cfg, not /boot/grub/i386-pc/grub.cfg.
If you are trying to run grub-mkconfig in a chroot or systemd-nspawn container, you might notice that it does not work: grub-probe: error: failed to get canonical path of /dev/sdaX. In this case, try using arch-chroot as described in the BBS post.
Use the grub-mkconfig tool to generate /boot/grub/grub.cfg:
```
grub-mkconfig -o /boot/grub/grub.cfg
```
By default the generation scripts automatically add menu entries for all installed Arch Linux kernels to the generated configuration.

Tip:
After installing or removing a kernel, you just need to re-run the above grub-mkconfig command.
For tips on managing multiple GRUB entries, for example when using both linux and linux-lts kernels, see /Tips and tricks#Multiple entries.
To automatically add entries for other installed operating systems, see #Detecting other operating systems.

You can add additional custom menu entries by editing /etc/grub.d/40_custom and re-generating /boot/grub/grub.cfg. Or you can create /boot/grub/custom.cfg and add them there. Changes to /boot/grub/custom.cfg do not require re-running grub-mkconfig, since /etc/grub.d/41_custom adds the necessary source statement to the generated configuration file.

Tip: /etc/grub.d/40_custom can be used as a template to create /etc/grub.d/nn_custom, where nn defines the precedence, indicating the order the script is executed. The order scripts are executed determine the placement in the GRUB boot menu. nn should be greater than 06 to ensure necessary scripts are executed first.
See #Boot menu entry examples for custom menu entry examples.

**Detecting other operating systems**

To have grub-mkconfig search for other installed systems and automatically add them to the menu, install the os-prober package and mount the partitions from which the other systems boot. Then re-run grub-mkconfig. If you get the following output: Warning: os-prober will not be executed to detect other bootable partitions then edit /etc/default/grub and add/uncomment:

GRUB_DISABLE_OS_PROBER=false
Then try again.

Note:
The exact mount point does not matter, os-prober reads the mtab to identify places to search for bootable entries.
Remember to mount the partitions each time you run grub-mkconfig in order to include the other operating systems every time.
os-prober might not work properly when run in a chroot. Try again after rebooting into the system if you experience this.
Tip: You might also want GRUB to remember the last chosen boot entry, see /Tips and tricks#Recall previous entry.

**Setting the top-level menu entry**

By default, grub-mkconfig sorts the included kernels using sort -V and uses the first kernel in that list as the top-level entry. This means that, for example, since /boot/vmlinuz-linux-lts is sorted before /boot/vmlinuz-linux, if you have both linux-lts and linux installed, the LTS kernel will be the top-level menu entry, which may not be desirable. This can be overridden by specifying GRUB_TOP_LEVEL="path_to_kernel" in /etc/default/grub. For example, to make the regular kernel be the top-level menu entry, you can use GRUB_TOP_LEVEL="/boot/vmlinuz-linux".

# Configure ZFS

**Note: We are under a chroot so don't try to start systemd services, just enable them.**

**Skip all steps in ZFS#zfs-mount-generator except enabling zfs-zed.service. ~~We will populate the cache later.~~**

**Unlock/Mount at boot time: systemd**

It is possible to automatically unlock a pool dataset on boot time by using a systemd unit. For example create the following service to unlock any specific dataset:

/etc/systemd/system/zfs-load-key@pool0-dataset0&period;service
```
[Unit]
Description=Load %I encryption keys
Before=systemd-user-sessions.service zfs-mount.service
After=zfs-import.target
Requires=zfs-import.target
DefaultDependencies=no

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/bash -c 'until (systemd-ask-password "Encrypted ZFS password for %I" --no-tty | zfs load-key %I); do echo "Try again!"; done'

[Install]
WantedBy=zfs-mount.service
```
Enable/start the service for each encrypted dataset, (e.g. zfs-load-key@pool0-dataset0&period;service). Note the use of -, which is an escaped / in systemd unit definitions. See systemd-escape(1) for more info. 

**Importing pools at startup**

ZFS provides systemd services for automatically importing pools and targets for other units to determine the state of ZFS initialization. These are:

| service | description |
|---|---|
| zfs.target | which is reached when all ZFS services completes |
| zfs-import.target | which is reached when ZFS pools finish importing |
| zfs-volumes.target | which is reached when zvols all appear under /dev |
| zfs-import-scan.service | which imports pools by scanning for devices using libblkid |
| zfs-volume-wait.service | which waits for all zvols to be available. |
```
systemctl enable zfs-zed.service
systemctl enable NetworkManager
systemctl enable dhcpcd
```
**zfs-import-scan**

zfs-import-scan.service uses zpool import's default logic of scanning devices using blkid, this means **no zpool.cache files are needed. This is the recommended method since zpool.cache is deprecated.**

**It is important to make sure non of your pools are imported with the cachefile option enabled since zfs-import-scan.service will not start if zpool.cache exists and is not empty. You can achieve this by enabling the zfs_autoimport_disable option of the zfs module. You should also either remove the existing zpool.cache or setting cachefile to none for all imported pools when booting.**

**Automatically mounting filesystems**

The service zfs-import-scan.service will import the pools without mounting any filesystems. To also mount filesystems on startup there are 2 methods, depending on if your filesystems are configured using mountpoint=legacy or not. If your filesystems are configured with a mix of legacy mount and non-legacy mount you'll need to use both methods.

**zfs-mount-generator**

If your filesystems use non-legacy mount, it is recommended to use zfs-mount-generator, which is a systemd.generator(7) that generates systemd mount units for all filesystems of imported zfs pools with the property canmount=on to mount filesystems on boot. By default though zfs-mount-generator won't do anything since it requires zfs list caches. You need to:

Enable and start zfs-zed.service
Create the /etc/zfs/zfs-list.cache directory.
Create empty files named after your pools in /etc/zfs/zfs-list.cache. Zed will only update the list of filesystems if the file for the pool already exists and is writable.
```
touch /etc/zfs/zfs-list.cache/<pool-name>
```
Check the contents of /etc/zfs/zfs-list.cache/<pool-name>. If it is empty, change the canmount property of any of your ZFS filesystem to emit ZFS events to trigger zed by running:
```
zfs set canmount=off <pool>/<dataset>
zfs inherit canmount <pool>/<dataset>
```

**fstab**

If your filesystem uses legacy mount, then you should specify the mountpoint in the fstab file. The device field should be the name (full path) of your filesystem and the dump and fsck fields should be left as 0.
- currently not using legacy mount


**Creating a hostid file**

While not strictly necessary, it is usually a good idea to create a /etc/hostid file:
```
zgenhostid $(hostid)
```

**Set up the initrd using zfs hook**

The zfs hook is the only option when using the default busybox based initrd.

To configure the zfs hook, simply add zfs before the filesystems hook in your mkinitcpio.conf(5) **Due to zfs root also remove the fsck hook.** (you can leave fsck alone if you want, I just wanted to get rid of the error message it produces)
```
nano /etc/mkinitcpio.conf
```
```
mkinitcpio -P
```
~~**Update /etc/default/grub kernel parameter**~~

~~nano /etc/default/grub~~

~~Add root=ZFS=<pool/dataset> to GRUB_CMDLINE_LINUX=~~

~~grub-mkconfig -o /boot/grub/grub.cfg~~

Possible syntax of kernel parameters are:

| parameter | description |
|---|---|
| root=zfs | determines the root filesystem using the bootfs property |
| root=ZFS=<pool/dataset> | which uses a pool or a dataset as root. When a pool is specified, the root filesystem is determined based on the mountpoint property |
| zfs=auto | same effect as root=zfs |
| zfs=<pool/dataset> | same effect as root=ZFS=<pool/dataset> |

Additionally, the following kernel parameters can be set to adjust the behavior of the initrd:

zfs_force=1 makes the zpool import command use the -f flag

zfs_wait=<seconds> waits for the devices to show up before running zpool import

See #Initrd tools to configure the initrd generator of your choice. Don't forget to regenerate initrd via:
```
mkinitcpio -P 
```
**Exit the chroot.**
```
exit
```

**Unmount, export and reboot**

Unmount all mounted filesystems (assuming the altroot is /mnt):
```
umount -R /mnt
```
Export all pools:
```
zfs set mountpoint=none <zpool>
zpool export -a
```
Reboot:
```
reboot
```

**Post-installation**

**Don't login as root**

See General recommendations for system management directions and post-installation tutorials (like creating unprivileged user accounts, setting up a graphical user interface, sound or a touchpad).

For a list of applications that may be of interest, see List of applications.

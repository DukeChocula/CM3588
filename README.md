
# FriendlyElec CM3588
<img src="https://wiki.friendlyelec.com/wiki/images/c/cc/CM3588SDK-A01.jpg" width="400">

The [FriendlyElec (NanoPC) CM3588](https://www.friendlyelec.com/index.php?route=product/product&product_id=294) is a RK3588 based solution with 4/8/16GB LPDDR4x memory and 0/64GB eMMC flash storage. The inital carrier board released with the CM3588 features 4x M.2 NVMe SSD slots (PCIe 3.0 x1 each) and a 2.5gbps RJ45 port, making it an attractive option for a low powered silent NAS at $130/$145/$174 depending on which RAM configuration you purchase. 

I ordered this in response to the [LTT video](https://www.youtube.com/watch?v=QsM6b5yix0U). I have seen a few people struggling to configure these, and figured I would offer some basic guides in order to get those who need some assistance on getting their NAS up and going.

I will be using the precompiled Debian 12 installer found [here.](https://drive.google.com/drive/folders/1FoBbP_nPkMehwBj4wHwsbRU-QGjEdeEP) ( 01_Official images > 02_SD-to-eMMC images > rk3588-eflasher-debian-bookworm-core-6.1-arm64-xxxxxxxx.img.gz)

The [FriendlyElec Wiki](https://wiki.friendlyelec.com/wiki/index.php/CM3588) is actually pretty good, but it can feel like a wall of text/commands to run and can be overwhelming to a new user.

Want to power this over POE+? I used [this adapter](https://www.amazon.com/dp/B09CYGW46K), and verified it supplies up to 25 watts and passes 2.5gbps. 

Typical power consumption with 4 Micron 2300 NVMe drives and (2) Noctua NF-A4x10 fans @ 5V: 
* idle = ~5w idle
* "typical" load = ~7-15w
* stress testing CPU and SSDs = ~20-23w

I have found that transfering at linerate (2.5gbps) uses about 35% CPU and ~15-20w in my testing.

I remixed [sgofferj's CM3588-NAS-case](https://github.com/sgofferj/CM3588-NAS-case) to accomadate the case fan screws (M5.5) that come with the Noctua fans. You can find that [here.](https://makerworld.com/en/models/469663)

## Installing an OS
Since I bought a 8GB RAM/64GB eMMC model, I will be using an SD to eMMC install image for this guide. You will need a MicroSD card that is 8GB or larger.

1) Download Debian 12 Bookworm Core from [here.](https://drive.google.com/drive/folders/1FoBbP_nPkMehwBj4wHwsbRU-QGjEdeEP) ( 01_Official images > 02_SD-to-eMMC images > rk3588-eflasher-debian-bookworm-core-6.1-arm64-xxxxxxxx.img.gz)
2) Use a tool like [7Zip](https://www.7-zip.org/), to extract the .img file
3) Use [Balena Etcher](https://etcher.balena.io/) to write the image to the SD card.
4) Install the SD card into the CM3588 NAS, then power the unit on.
5) If you use one of the HDMI out ports, you can track the progress. Otherwise you can wait ~5 minutes (mine consistently takes around 80 seconds to install).
6) Power down the unit, then power it back on.
7) If you have HDMI hooked up, you should be prompted at the main login screen. If you want to use SSH, scan your network or check your router/DHCP server for the device's IP address.

## Configuring Debian 12
There are 2 accounts by default.
* root/fa
* pi/pi (in the sudoers group)

In this guide we will be:
* [Changing root password](#changing-root-password)
* [Deleting default/Pi user](#delete-default-user)
* [Creating a new user](#create-a-new-user)
* [Disabling root SSH logins](#disabling-root-ssh-logins)
* [Reconfiguring Apt sources](#reconfigure-apt-sources)
* [Compiling linux-headers for DKMS](#compiling-linux-headers-for-dkms-dynamic-kernel-module-support)
* [Installing ZFS](#installing-zfs)
* [Other guides](#other-useful-links-for-configuring-debian)

Use your favorite tool to SSH into the unit.

### Changing root password

```bash
passwd
```
Enter a new password, and confirm.

### Delete default user
We are going to create our own user, so we don't need this one.

```bash
deluser pi
```

### Create a new user 
I am using nas in this example, but feel free to name it whatever you want, then we will add it to sudoers group
```bash
useradd nas
```
```bash
passwd nas
```
```bash
usermod -aG sudo nas
```

### Disabling root SSH logins
This is for security, as the username is always root and the access rights are unlimited.

```bash
nano /etc/ssh/sshd_config
```
Go to line 33, change `PermitRootLogin yes` to `PermitRootLogin no`.
```bash
systemctl restart sshd
exit
```
SSH back in as your newly created user

### Reconfigure Apt sources
By default the sources file comes with a mirror that is based out of China. While this is _fine_, it would be much faster if you used the local Debian apt sources. So we will change them back. I have commented them out for now.
```bash
mv /etc/apt/sources.list /etc/apt/sources.list.old
nano /etc/apt/sources.list
```

```bash
#deb https://mirrors.aliyun.com/debian bookworm main non-free contrib
#deb-src https://mirrors.aliyun.com/debian bookworm main non-free contrib
#deb https://mirrors.aliyun.com/debian-security bookworm-security main
#deb-src https://mirrors.aliyun.com/debian-security bookworm-security main
#deb https://mirrors.aliyun.com/debian bookworm-backports main non-free contrib
#deb-src https://mirrors.aliyun.com/debian bookworm-backports main non-free contrib

deb http://deb.debian.org/debian bookworm main non-free-firmware contrib
deb-src http://deb.debian.org/debian bookworm main non-free-firmware contrib

deb http://deb.debian.org/debian-security/ bookworm-security main non-free-firmware contrib
deb-src http://deb.debian.org/debian-security/ bookworm-security main non-free-firmware contrib

deb http://deb.debian.org/debian bookworm-updates main non-free-firmware contrib
deb-src http://deb.debian.org/debian bookworm-updates main non-free-firmware contrib

deb http://deb.debian.org/debian bookworm-backports main non-free-firmware contrib
deb-src http://deb.debian.org/debian bookworm-backports main non-free-firmware contrib
```

### Compiling linux-headers for DKMS (Dynamic Kernel Module Support)
In order to install ZFS, we first need to install linux headers for our kernel to add DKMS support.

Luckily, pre-compiled headers can already be found in /opt/archives

```bash
sudo -i
dpkg -i /opt/archives/linux-headers-6.1.57_6.1.57-11_arm64.deb
```

#### Installing ZFS
Now that we have Linux headers, we can use apt to install ZFS

```bash
sudo apt install zfs-dkms
sudo apt install zfsutils-linux
```

Now that ZFS is installed, we will try to run a command to test that ZFS and DKMS are actually functional.

```bash
zpool status
```

If this works, it will report that we have 0 pools. 
If you get an error that says: "The ZFS modules are not loaded. Try running '/sbin/modprobe zfs' as root to load them", that means DKMS isn't working properly and you likely missed a step up above or it failed to compile, which should have given you an error.

Now that we have ZFS working, we can create our ZFS pool. You have a few options, depending on how much redundancy or performance you want. I would recommend the following options, although there are a few more. You will have to check the [ZFS documenation](https://openzfs.github.io/openzfs-docs/) for any other array types.

RAIDZ1 (RAID 5)
```bash
zpool create mypool raidz nvme0n1 nvme1n1 nvme2n1 nvme3n1
```
Mirror (RAID 1)
```bash
zpool create mypool mirror nvme0n1 nvme1n1 nvme2n1 nvme3n1
```
Striped Mirror (RAID 10)
```bash
zpool create mypool mirror nvme0n1 nvme1n1 mirror nvme2n1 nvme3n1
```

Now if we run `zpool status` we should get something like this:

```bash
$zpool status
  pool: mypool
 state: ONLINE
config:

        NAME         STATE     READ WRITE CKSUM
        mypool       ONLINE       0     0     0
          raidz1-0   ONLINE       0     0     0
            nvme0n1  ONLINE       0     0     0
            nvme1n1  ONLINE       0     0     0
            nvme2n1  ONLINE       0     0     0
            nvme3n1  ONLINE       0     0     0

errors: No known data errors
```

The array will be mounted at /mypool (or whatever you named your pool). You can move the mountpoint to a different location:

```bash
zfs set mountpoint=/mnt/storage mypool
```

Since this is all SSD array, I recommend enabling autotrim. Trim marks the invalid data and tells the SSD to ignore it during the garbage collection process, allowing your SSD to do some cleanup on its end.

```bash
zpool set autotrim=on mypool
```

Once this is complete, you will be ready to install whatever you would like.

##  Other useful links for configuring Debian
* [Configuring automatic ZFS scrubs](https://brismuth.com/scheduling-automated-zfs-scrubs-9b2b452e08a4)
* [Setting a static IP](https://wiki.debian.org/NetworkConfiguration)
* [Configuring Debian to install security updates automatically](https://wiki.debian.org/UnattendedUpgrades)
* [Installing Docker](https://docs.docker.com/engine/install/debian/)
* [Installing Yacht (Docker WebUI management)](https://yacht.sh/docs/Installation/Install/)
* [Installing Samba (SMB) server](https://serverspace.io/support/help/configuring-samba-on-debian/)

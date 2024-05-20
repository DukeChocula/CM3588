
# FriendlyElec CM3588
<img src="https://wiki.friendlyelec.com/wiki/images/c/cc/CM3588SDK-A01.jpg" width="400">


I ordered this in response to the [LTT video](https://www.youtube.com/watch?v=QsM6b5yix0U). I have seen a few people struggling to configure these, and figured I would offer some basic guides in order to get those who need some assistance on getting their NAS up and going.

The [FriendlyElec (NanoPC) CM3588](https://www.friendlyelec.com/index.php?route=product/product&product_id=294) is a RK3588 based solution with 4/8/16GB LPDDR4x memory and 0/64GB eMMC flash storage. The inital carrier board released with the CM3588 features 4x M.2 NVMe SSD slots (PCIe 3.0 x1 each) and a 2.5gbps RJ45 port, making it an attractive option for a low powered silent NAS at $130/$145/$174 depending on which RAM configuration you purchase. 

I will be using the precompiled Debian 12 installer founder [Here](https://drive.google.com/drive/folders/1pRbY9IMKlwChIOfoE6I9tT6dZd5T8p2E)

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

1) Download Debian 12 Bookworm Core from [here.](https://drive.google.com/drive/folders/1pRbY9IMKlwChIOfoE6I9tT6dZd5T8p2E) (rk3588-eflasher-debian-bookworm-core-6.1-arm64-20240511.img.gz)
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

deb http://deb.debian.org/debian bookworm main non-free-firmware
deb-src http://deb.debian.org/debian bookworm main non-free-firmware

deb http://deb.debian.org/debian-security/ bookworm-security main non-free-firmware
deb-src http://deb.debian.org/debian-security/ bookworm-security main non-free-firmware

deb http://deb.debian.org/debian bookworm-updates main non-free-firmware
deb-src http://deb.debian.org/debian bookworm-updates main non-free-firmware

deb http://deb.debian.org/debian bookworm-backports main non-free-firmware
deb-src http://deb.debian.org/debian bookworm-backports main non-free-firmware
```

### Compiling linux-headers for DKMS (Dynamic Kernel Module Support)
In order to install ZFS, we need to compile linux headers for our kernel to add DKMS support.

```bash
sudo -i
git clone https://github.com/friendlyarm/sd-fuse_rk3588.git --single-branch -b kernel-6.1.y
cd sd-fuse_rk3588
git clone https://github.com/friendlyarm/kernel-rockchip --depth 1 -b nanopi6-v6.1.y kernel-rk3588
```

Download the source code for the kernel [here](https://drive.google.com/drive/folders/1WFnZzNtQJLMDKYKh8rxIO4YuNrDur2qM) (debian-bookworm-core-arm64-images.tgz), then use a tool like WinSCP to copy this to the sd-fuse-rk3588 folder. While you _technically_ can skip this step (the script will download the source code for you), it comes from a server in China where you have to download ~550MB @ 200kbps

Compiling the linux headers will take awhile (~50 minutes)
```bash
tar xvzf debian-bookworm-core-arm64-images.tgz
MK_HEADERS_DEB=1 BUILD_THIRD_PARTY_DRIVER=0 KERNEL_SRC=$PWD/kernel-rk3588 ./build-kernel.sh debian-bookworm-core-arm64
cd out/
dpkg -i linux-headers-6.1.57.deb
exit
```

#### Installing ZFS
Now that we have Linux headers, we can use apt to install ZFS

```bash
sudo apt install zfsutils-linux
```

Now that ZFS is installed, we will try to run a command to test that ZFS and DKMS are actually functional.

```bash
zpool status
```

If this works, it will report that we have 0 pools. 
If you get an error that says: "The ZFS modules are not loaded. Try running '/sbin/modprobe zfs' as root to load them", that means DKMS isn't working properly.

Now that we have ZFS working, we can create out ZFS pool. You have a few options, depending on how much redundancy or performance you want. I would recommend the following options, although there are a few more. You will have to check the [ZFS documenation](https://openzfs.github.io/openzfs-docs/) for any other array types.

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

The array will be mounted at /POOLNAME

Since this is all SSD array, I recommend enabling autotrim. Trim marks the invalid data and tells the SSD to ignore it during the garbage collection process.

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

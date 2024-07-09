# Simple Wireless Proxmox Configuration

Searching for information on Proxmox configuration can often lead to vague, outdated, conflicting or sometimes outright incorrect results. Given the underlying complexity of the operations the system is expected to perform, this should not be surprising. In many instances, even though configuration instructions may be technically correct, the number of component steps that require detailed research to overcome undocumented issues can cause significant distraction. This is a long standing feature of many Linux distributions and applications, and a contributor to the reluctance of general computer users to adopt Linux as their primary operating system.

My intended use case for Proxmox is a small homelab for developing software programs for IoT devices, such as security cameras. I want to have a subnet dedicated to the cameras, and to have a private git server that can be used as a software repository. I also have an issue with internet access in that it is impractical for my purposes to use an ethernet cable connection from the Proxmox server to the home router. I need to have a wireless connection from the Proxmox server to the wireless router for access to the internet, which must be available for use by LXCs and VMs created in Proxmox.

While this might seem like a reasonable configuration to the casual observer, it is anything but. Proxmox does not support wireless connection out of the box, and giving access to the wireless interface to VMs within Proxmox is not a well documented configuration. There are also a number of quirks in the Proxmox system that can only be discovered through trial and error, so experience with the system under field operating conditions is key to a robust configuration.

I made these notes to myself as I went through the exercise of setting of my system, so that I could re-build or replicate it in the event that I needed to do such things. My intention is to document every step of the process in detail, so that my future self, who has forgotten the path taken to overcome roadblocks, is able to re-construct the configuration easily. If someone who is working on a similar issue finds these notes, I hope that they are helpful.

# Create Bootdisk for Installing Proxmox

Proxmox is an Operating System based on Debian Linux. When you install Proxmox you are installing an Operating System as opposed to an application, so it is basically taking over the entire machine. It is possible to run Proxmox as a kind of application running under Debian, but this is an advanced topic not covered here. To install Proxmox, [Download the ISO](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso/proxmox-ve-8-2-iso-installer) and write it to USB using an etcher such as [Balena Etcher](https://etcher.balena.io/) on Linux, or [Rufus](https://rufus.ie/en/) on Windows.

Using Rufus on Windows is very straightforward, the Rufus download is an executable file and can be launched directly by double clicking from the file manager. Balena Etcher however requires additional steps for proper operation. Without exhaustive testing, I was unable to get the latest version, 1.19.21 to work. The version that worked for me on Ubuntu 24.04 was 1.18.11, using the AppImage style file. The [FUSE utility](https://github.com/AppImage/AppImageKit/wiki/FUSE) installation is required. Please read and follow carefully the FUSE instructions as shown in the link above. It is necessary to set the AppImage file permissions to executable. When launching the program, it is necessary to use the --no-sandbox option.

    chmod +x balenaEtcher-1.18.11-x64.AppImage
    ./balenaEtcher-1.18.11-x64.AppImage --no-sandbox

# Install Proxmox

Note that Proxmox only supports HDMI video output, so a Display Port output will not work. The host should be connected to an HDMI monitor and a keyboard and mouse during configuration. Plug the USB into the machine and power on, using the appropriate BIOS hot key to select the USB boot option. The setup will ask for timezone questions and such, then will ask for the network configuration. This is the important part. This document assumes that you are setting up a sub net with the Proxmox host. Use the wired connection for the configuration, the wireless network interface will not work, even though it is listed as on option. You will need to know the IP address of your wireless router, in all likelihood, it will be 192.168.1.1. It is suggested that you use 10.1.1.1 for the wired network address of the Proxmox host, so that the IP addresses on the wired subnet are easily disinguishable from other IP address on your wireless home network. Use the 10.1.1.1 address for the wired network gateway and use the wireless router address for the DNS server. Set the password and the system will reboot when the installation is complete. The system will prompt you to remove the installation media during reboot. 

You will now need another machine (client) to connect to the Proxmox host on its network. This will require a network hub to which both computers may be attached. Use the ethernet port of the client to connect a wire to the hub shared with the host. Set the IP address of the client to something within the subnet range of the host e.g. 10.1.1.2. You will now be able to connect to the host using ssh.

    ssh root@10.1.1.1

You will receive a prompt asking if you want to trust the host, say yes. After entering your password, you should have a terminal prompt,

# Wireless Configuration

To enable the host wireless connection, use the [Debian package iwd](https://wiki.debian.org/NetworkManager/iwd). This package has a dependency on another package, the Embedded Linux library (libell0). Download the [iwd deb file](https://packages.debian.org/bookworm/iwd) and the [libell0 deb file](https://packages.debian.org/stable/libs/libell0) and copy them to USB. Insert the USB into the host system, mount the USB and install the libell0 deb package first, then the iwd deb package, using the following instructions.

<b>Mount USB Drive</b>

Use the command fdisk -l to show drives in the system. You are looking for the USB which will look something like:

    Disk /dev/sda: 28.91 GiB, 31042043904 bytes, 60628992 sectors
    Disk model: USB 3.2.1 FD    
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x1cf2405c

    Device    Boot Start       End  Sectors  Size  Id  Type
    /dev/sda1       8064  60628991 60620928 28.9G   C  W95 FAT32 (LBA)

Notice that there are two entries for the usb, disk and device. Device is the better bet when mounting the USB.

<b>Create Mount Point</b>

You will need a file location for mounting the USB within the file system, so create a directory something like:

    mkdir /media/usb

<b>Mount the USB Drive</b>

    mount /dev/sda1 /media/usb

You will now be able to access the USB drive as you would any normal file on the system.

<b>Install the .deb Packages</b>

    apt install /media/usb/libell0_0.56-3_amd64.deb
    apt install /media/usb/iwd_2.3-1+deb12u1_amd64.deb

Your package names may vary slightly based upon the current version of the package.

<b>Unmount the USB Drive when finished</b>

    umount /media/usb

This step is necessary for a graceful shutdown of the USB connection in order to avoid damage to the files.

<b>Run the iwd Service</b>

The iwd service can now be initialized with the following command:

    systemctl --now enable iwd

In order for the service to configure the wireless connection, it is necessary to edit the configuration file. These instructions show the use of nano as the text editor. If you are new to nano, it is a simple program that can edit files from the console. The important commands to remember are ctl+O followed by enter to save the file, and ctl+X to exit the editor. Starting nano as shown below will open the file. This file should already exist on the system, but please note that nano will create a blank file if it does not exist already.

    nano /etc/iwd/main.conf

At the top of the file is the ```[General]``` section. Remove the hashmark in front of the following line to enable:

    EnableNetworkConfiguration=true

Now restart the iwd service

    service iwd restart

<b>Configure the Wireless Connection</b>

Before continuing, make note of the name of the wireless interface. Use the command

    ip a

to show a list of the available interfaces. The wireless interface will most likely be named ```wlan0```

To begin configuration, use the command

    iwctl

This will change the command prompt to ```[iwd]```, which can be exited with the command ```quit```. Under this prompt, use the following command to show a list of available wireless networks, which should look familiar.

    station wlan0 get-networks

The network to which you intend to connect should be shown in the list. Note that the screen will blink as the network list is refreshed. Once you have identified the desired network, use the ```quit``` command to exit the utility and stop refreshing. Now you will restart the command prompt and use the following command to connect to the network. You will be prompted to enter the WiFi network password.

    iwctl
    station wlan0 connect NAME_OF_YOUR_WIFI_NETWORK

At this point, you should be able to ping the WiFi router. If successful, you should be able ping to the internet. For example, the following commands should elicit correct responses:

    ping 192.168.1.1
    ping 8.8.8.8
    ping google.com

If the first command fails, the wireless connection is not working. If first succeeds and the second fails, the WiFi router is not routing your ping out to the internet. If the first two succeed and the third fails, the DNS is not resolving. In this last case, check your ```/etc/resolv.conf``` file has the WiFi router address as the nameserver.

Upon completion of the wireless configuration, you can update the system by using the apt commands:

    apt update
    apt upgrade

# Packet Forwarding For the Wireless Connection

In order for other devices or virtual machines on the wired subnet to be able to communicate to the internet through the wireless connection on the proxmox host, it is necessary to forward packets through the host, similar to the way a router functions. This can be done using [nftables](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page) on Linux.

The first step is to enable forwarding by editing the ```/etc/sysctl.conf``` file on the proxmox host. Uncomment the following line by deleting that hashmark

    net.ipv4.ip_forward=1

Save the file and restart the utility with the following command:

    sysctl -p

You should receive an acknowledgement that the forwarding variable has been set to 1. The following nftables configuration script will implement packet forwarding on the host. The basic idea is that you are setting up filter and NAT tables that will capture packets on one of the network interfaces and forward them through it's gateway to the other connected network. During the forwarding, the script will translate the addresses from one network to the other, and will masquerade the packet as belonging to the host to the other network, so that the packets are accepted as valid by the receiving network. Edit the file ```/etc/nftables.conf``` to appear as follows:

```
#!/usr/sbin/nft -f

flush ruleset
table inet filter {
        chain input {
                type filter hook input priority filter; policy accept;
        }

        chain forward {
                type filter hook forward priority filter; policy accept;
        }

        chain output {
                type filter hook output priority filter; policy accept;
        }
}
table inet nat {
        chain prerouting {
                type nat hook prerouting priority dstnat; policy accept;
        }

        chain postrouting {
                type nat hook postrouting priority srcnat; policy accept;
                ip saddr 10.1.1.0/24 masquerade
        }
}   
```

When you are done with the configuration file, the command

    systemctl enable nftables --now

Will initialize the configuration. You can verify the effect of the script by using the ```nft list ruleset``` command, the output of which should have an appearance very similar. The configuration should persist through a reboot.

 # Configure DHCP Server

Although devices and virtual machines may be configured with static IP addresses in order to communicate with the network, this is often an impractical approach as the number of devices grows. Additionally, some devices may require DHCP in order to be initialized. A DHCP server can provide the ability to assign IP addresses automatically, reducing the administrative burden. One approach to this problem could be to add a router to the subnet, however, this introduces another physical device that has to be managed, and many modern routers available for retail consumers are not easily configured for this type of application. An easier method is to add a DHCP server to the Proxmox host so that it can provide IP address management. In the following example, an LXC will be created and configured as a DHCP server.

 <b>Download Template and Configure LXC</b>

LXC templates are available for download. To view these, use a pve console to run the command ```pveam available```. This will show a list of templates. This example will use the ```debian-12-standard_12.2-1_amd64.tar.zst``` from the system section of templates as a starting point. This is a good choice for a nice and stable LXC without a lot of frills. Please note that LXC, container and CT are terminology that refer to the same thing, and that the Proxmox host is often referred to as pve (Proxmox Virtual Environment). To download the LXC template, use the following command from the pve console: 

    pveam download debian-12-standard_12.2-1_amd64.tar.zst 

Use the Create CT button, set the Hostname and password, then click Next to go to the Template and and use the dropdown box to select the template that was downloaded. The defaults can be accepted for the Disks, CPU and Memory tabs. When you get to the Network Tab, note the name of the network, which will probably be eth0. Set a static IP, such as 10.1.1.3/24 and 10.1.1.1 as the Gateway. On the DNS tab, set the DNS servers to your wireless router as above e.g. 192.168.1.1.

Once the LXC has been created, start the LXC and open the container console to verify the network connections using the ping tests as shown in the wireless configuration instructions above. These commands should succeed with the same results as if they were sent from the host itself. If the host is working properly, but the container does not connect to the internet, there may be an issue with the packet forwarding or perhaps the LXC gateway. 

<b>Install and Configure DHCP Server</b>

If all is running as expected, update the system, then install kea, the DHCP server: 

    apt update
    apt upgrade
    apt install kea
    
The kea server is configured in a plain text file located at ```/etc/kea/kea-dhcp4.conf```. The file should be edited to look something like that shown below. Near the top of the file is the ```interfaces: ``` section, which should be populated with the network interface name assigned to the container at creation. The ```pool: ``` section allocates a range of IPs that are available for assignment by the DHCP server. Blocks of IPs not covered in this range are available for devices with static IPs. The ```data``` section of the ```routers``` is set to the Proxmox host IP address. The ```data``` section of the ```domain-name-server``` should be populated with the IP address of the wireless router as has been used before. A recommended way of implmenting this configuration in the file is to navigate into the ```/etc/kea``` directory, move the existing kea-dhcp4.conf file to a backup, then open a nano editor and use copy and paste to fill it with the contents shown below. You can then make edits easily as needed.

```
{
"Dhcp4": {
    "interfaces-config": {
    "interfaces": [ "eth0" ]
    },
    "control-socket": {
        "socket-type": "unix",
        "socket-name": "/run/kea/kea4-ctrl-socket"
    },
    "lease-database": {
        "type": "memfile",
        "lfc-interval": 3600
    },
    "valid-lifetime": 600,
    "max-valid-lifetime": 7200,
    "subnet4": [
    {
        "id": 1,
        "subnet": "10.1.1.0/24",
        "pools": [
        {
            "pool": "10.1.1.64 - 10.1.1.242"
        }
        ],
        "option-data": [
        {
            "name": "routers",
            "data": "10.1.1.1"
        },
        {
            "name": "domain-name-servers",
            "data": "192.168.1.1"
        },
        {
            "name": "domain-name",
            "data": "mydomain.example"
        }
        ]
    }
    ]
}
}
```

Once the configuration file has been finalized, you can start the kea server using the command:

    systemctl enable kea-dhcp4-server --now

This will start the server immediately and configure it to start on boot. You should set the container Options Start at boot in Proxmox as well so the the container itself will start automatically when the Proxmox host is booted. In order to check the status of the DHCP server, use the command:

    systemctl status kea-dhcp4-server

The output of this command should show that the server is enabled and that it has IP leases available and possibly allocated. If this is not the case, there is likely some issue with the configuration file. Once everything is set up, it is a good idea to reboot the Proxmox host to insure that everything starts up correctly on boot.

# Adding Bulk Storage to PVE

This is an optional step, but one that will provide benefits for most environments. As Proxmox is an operating system, it will occupy the main boot drive of the host computer. In many cases, this will be a faster NVME drive, which may be limited in capacity. Bulk storage can provide a larger capacity drive that is somewhat slower, but is used less often, so speed would be a secondary consideration. This is useful for backups and the storage of large files. 

In the case of Proxmox, virtual machines are created using .iso files, which can be several GB in size. It can be convenient to store these large files on a secondary HDD, and configure access through the PVE so that they may be deployed easily.

There are many ways to add bulk storage to a system, including NFS, a NAS server, a second NVME drive or a SATA HDD. In this example, a SATA HDD is configured, although the steps would be similar for NVME. In the case of NFS or NAS, the drive partition and format will have already been performed, so you could skip to the PVE integration once the drive is mounted.

<b>Partition and Format the HDD</b>

Assuming the physical drive has been connected to the host hardware, use the ```fdisk -l``` command to show the devices connected to the host. You are looking for the target HDD, which will probably look something like this:

    Disk /dev/sda: 1.82 TiB, 2000398934016 bytes, 3907029168 sectors
    Disk model: ST2000LM015-2E81
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 4096 bytes
    I/O size (minimum/optimal): 4096 bytes / 4096 bytes
    Disklabel type: gpt
    Disk identifier: 923E88EA-2D22-465B-A802-286F5C0AC4BF

If the drive has already been partitioned and formatted with the ext4 format, you would also see something like this in the fdisk output:

    Device     Start        End    Sectors  Size Type
    /dev/sda1   2048 3907028991 3907026944  1.8T Linux filesystem

It is important to note that Proxmox supports the Linux filesystem. If the partition has been formatted otherwise, it is unlikely to work properly. It is possible in this situation to create new partitions using unoccupied areas of the disk, but this is a more advanced configuration not covered here. For our purposes, the drive will have a single Linux partition that spans the entire disk. If there is important data existing on the drive and it is not formatted for Linux, please consider using another drive that can be wiped clean.

To create the partition, it is recommended to use ```gdisk```, which has more capabilities than the traditional fdisk. In the example above, the following command will start the gdisk interactive prompt:

    gdisk /dev/sda

The command will respond with something like:

    GPT fdisk (gdisk) version 1.0.9

    Partition table scan:
    MBR: protective
    BSD: not present
    APM: not present
    GPT: present

    Found valid GPT with protective MBR; using GPT.

    Command (? for help): 

The available gdisk commands are shown below

    Command (? for help): ?
    b       back up GPT data to a file
    c       change a partition's name
    d       delete a partition
    i       show detailed information on a partition
    l       list known partition types
    n       add a new partition
    o       create a new empty GUID partition table (GPT)
    p       print the partition table
    q       quit without saving changes
    r       recovery and transformation options (experts only)
    s       sort partitions
    t       change a partition's type code
    v       verify disk
    w       write table to disk and exit
    x       extra functionality (experts only)
    ?       print this menu

To start from a fresh disk, delete any existing partitions using the ```d``` command, then use the ```n``` command to create a new partition. If you accept all defaults, a single Linux partition spanning the entire disk will be created. Use the ```w``` command to commit the change to the drive. Once the partition has been created it should be visible using the ```lsblk``` command. Continuing with the example case above, the partition can now be formatted using the command:

    mkfs -t ext4 /dev/sda1

The effect of the format should now be visible using the ```lsblk -f``` command.

If all has gone well, it should now be possible to mount the drive. It is necessary to have a mount point in the host filesystem, which can be created such as:

    mkdir -p /mnt/hdd
    mount /dev/sda1 /mnt/hdd

To make this mount permanent, edit the ```/etc/fstab``` file and add an entry like so to the end:

    /dev/sda1 /mnt/hdd ext4 defaults 0 0

Now reboot the PVE host and verify that the mount is persistent by navigating to the /mnt/hdd directory and writing a sample file using nano.

<b>Integrate ISO Directory from Bulk Storage</b>

As you work with Proxmox and start creating virtual machines, you will start building a collection of iso files, which can be quite large. There is a way to integrate them into the PVE from a bulk storage device so that you don't have to dedicate a lot of space from you main drive to house them.

The first step is to assign a directory on the bulk storage in which to house the ISO files. The name of this directory has a specific requirement in that it should end with the path

    var/lib/vz/template/iso

This is done so the PVE is able to find the files. Copy the .iso files into this location, then use the Proxmox web interface to go to Datacenter Storage and add a new Directory. Set the ID to iso and the directory to the vz subdirectory created on the bulk storage device created above. Following the example of the SATA HDD presented above, this would be 

    /mnt/hdd/var/lib/vz

Additionally, you will need to use the Content dropdown box to specify ISO Image. Click the Add button, and the iso location should appear on the list in the Storage page. Now when you create a virtual machine, the Storage option should show iso and the ISO image drop down box should show the list of iso files on the bulk storage device.

# Install Gitea Server

Create an LXC with the debian template. A few cores are plenty, and it does not need a lot of memory or disk space. Set a fixed IP address, e.g. 10.1.1.4.  Alternatively, it could just be combined with the kea server on that virtual machine if desired. 

You will need to install git:

    apt install git

Now create a user for the server

```
adduser \
   --system \
   --shell /bin/bash \
   --gecos 'Git Version Control' \
   --group \
   --disabled-password \
   --home /home/git \
   git
```

Create a directory structure:

```
mkdir -p /var/lib/gitea/{custom,data,log}
chown -R git:git /var/lib/gitea/
chmod -R 750 /var/lib/gitea/
mkdir /etc/gitea
chown root:git /etc/gitea
chmod 770 /etc/gitea
```

Download the gitea executable to the machine. Note that the @version@ tag is replaced with the version desired, the most recent at the time of this writing is 1.22.0:

    wget -O /user/local/bin/gitea https://dl.gitea.com/gitea/@version@/gitea-@version@-linux-amd64
    chmod +x /usr/loca/bin/gitea

Copy this file to /etc/systemd/system/gitea.service

```
[Unit]
Description=Gitea (Git with a cup of tea)
After=network.target

[Service]
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
```
The server should be started with the command:

    systemctl enable gitea --now

Now launch a web browser tab to the gitea server at port 3000 and you should get an install page. Use the Database option SQLite3 for easiest configuration and click Install. You can now create a user account and you should be good to go.

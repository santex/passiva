## TP-Link MR3020

### OpenWRT Installation
<http://store.jpgottech.com/support/tp-link-mr3020-openwrt-flashing-guide/>

open MR3020 webinterface on http://192.168.0.254/ 
* username "admin"
* password "admin"

flash [OpenWrt firmware][1] like a regular firmware update

[1]: http://downloads.openwrt.org/attitude_adjustment/12.09/ar71xx/generic/openwrt-ar71xx-generic-tl-mr3020-v1-squashfs-factory.bin "OpenWrt downloads"

wait for progress bar to finish __twice__

from now on the OpenWrt router can be reached at http://192.168.1.1/

    $ telnet 192.168.1.1
    root@OpenWrt:~# passwd

set a password for root user, now telnet gets disabled an ssh enabled

    root@OpenWrt:~# reboot

#### Network configuration

connect to the TP-Link

    $ ssh -l root 192.168.1.1

NOTE: As most gateway (e.g. your home network) router’s IP address are 192.168.1.1, we will use 192.168.1.11 for the TP-Link MR3020.

    root@OpenWrt:~# vi /etc/config/network

The modified file should look like this:

    config interface 'loopback'
        option ifname 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'
    
    config interface 'lan'
        option ifname 'eth0'
        option type 'bridge'
        option proto 'static'
        option ipaddr '192.168.1.11'
        option netmask '255.255.255.0'
        option gateway '192.168.1.1'
        list dns '192.168.1.1'
        list dns '8.8.8.8'
    
    config interface 'wan'
        option ifname 'eth0.1'
        option proto 'dhcp'
        option gateway '192.168.1.1'
        option netmask '255.255.255.0'
        list dns '192.168.1.1'
        list dns '8.8.8.8'
    
    config interface 'wifi'
        option proto 'static'
        option ipaddr '192.168.2.1'
        option netmask '255.255.255.0'

#### Firewall settings

Backup firewall config file:

    root@OpenWrt:~# cp /etc/config/firewall /etc/config/firewall.bak

Open the firewall config file:

    root@OpenWrt:~# vi /etc/config/firewall

Modify first 23 lines to look like this. Leave the rest of the file alone.

    config defaults
            option syn_flood        '1'
            option input            'ACCEPT'
            option output           'ACCEPT'
            option forward          'ACCEPT'
    # Uncomment this line to disable ipv6 rules
    #       option disable_ipv6     1
    
    config zone
            option name             'lan'
            option network          'lan'
            option input            'ACCEPT'
            option output           'ACCEPT'
            option forward          'ACCEPT'
    
    config zone
            option name             'wan'
            option network          'wan'
            option input            'ACCEPT'
            option output           'ACCEPT'
            option forward          'ACCEPT'
            option masq             '1'
            option mtu_fix          '1'
    
    config zone
        option name     'wifi'
        option input        'ACCEPT'
        option output       'ACCEPT'
        option forward      'REJECT'
    
    config forwarding
        option src          'wifi'
        option dest         'wan'
    
    config forwarding
        option src      'wifi'
        option src      'lan'

We need to accept udp packets on port 68, see <https://dev.openwrt.org/ticket/4108>

#### Wireless settings

Enable wireless by modifying the wireless config:

    root@OpenWrt:~# vi /etc/config/wireless

Change Line 12 to:

    # option disabled 0
    config wifi-iface
        option device   radio0
        option network  wifi
        option mode     ap
        option ssid     OpenWrt
        option encryption none

#### USB support

Now You have OpenWRT on your TP-Link MR3020, now add USB support
Add USB support to OpenWrt by installing and enabling the following packages:

    root@OpenWrt:~# opkg update
    root@OpenWrt:~# opkg install kmod-usb-uhci
    root@OpenWrt:~# insmod uhci
    root@OpenWrt:~# opkg install kmod-usb-ohci
    root@OpenWrt:~# insmod usb-ohci

Filesystem on a USB stick (Rootfs on External Storage / extroot)
first we need to install basic USB support for EXT4 and FAT32 formatted USB sticks

    root@OpenWrt:~# opkg update
    root@OpenWrt:~# opkg install kmod-usb-storage block-mount kmod-fs-ext4
                    kmod-fs-vfat kmod-nls-cp437 kmod-nls-cp850
                    kmod-nls-iso8859-1 kmod-nls-iso8859-15 kmod-scsi-core
                    e2fsprogs fdisk

check if the USB stick was recognized

    root@OpenWrt:~# ls /dev/sd*

you should see one or more devices like sda1, sdb1, sdb2, ...

partition the USB stick

    root@OpenWrt:~# fdisk

    Command (m for help): m (displays actions)
    Command (m for help): p (display partition table)
    Command (m for help): d (delete partion - had only 1 on my stick)
    Command (m for help): n (new partition)
    Command action
    e extended
    p primary partition (1-4)
    p
    Partition number (1-4): 1
    First cylinder (1-621, default 1): RETURN
    Using default value 1
    Last cylinder or +size or +sizeM or +sizeK (1-nnnn, default nnn):
    Command (m for help): a (make partition bootable) RETURN
    Partition number (1-4): 1 Command (m for help): w (write to disk)

format USB stick with ext4 filesystem

    root@OpenWrt:~# mkfs.ext4 /dev/sda1

mount the USB stick and copy the flash /overlay to the USB stick

    root@OpenWrt:~# mkdir -p /mnt/usb
    root@OpenWrt:~# mount -t vfat /dev/sda1 /mnt/usb
    root@OpenWrt:~# tar -C /overlay -cvf - . | tar -C /mnt/usb -xvf -
    root@OpenWrt:~# vi /etc/config/fstab

the 'config mount' block should look like this

    config 'mount'
            option target   /overlay
            option device   /dev/sda1
            option fstype   ext4
            option options  rw,sync
            option enabled  1
            option enabled_fsck 0

restart the router and check if the filesystem

    root@OpenWrt:~# df -kh
    Filesystem           1K-blocks      Used Available Use% Mounted on
    rootfs                 1962212     65100   1798716   3% /
    /dev/root                 1536      1536         0 100% /rom
    tmpfs                    14600        72     14528   0% /tmp
    tmpfs                      512         0       512   0% /dev
    /dev/sda1              1962212     65100   1798716   3% /overlay
    overlayfs:/overlay     1962212     65100   1798716   3% /

__Note__: if the /root has a nigh % usage the overlay may not been in use.
If you want to start fresh and clean again:
Emtpy the /overlay folder containing all your configurations, installed packages, etc.

    # rm -rf /overlay/*

and upgrade the kernel / firmware

    # cd /tmp
    # wget <http://downloads.openwrt.org/snapshots/trunk/ar71xx/openwrt-ar71xx-generic-tl-mr3020-v1-squashfs-factory.bin>
    # sysupgrade -v /tmp/openwrt-ar71xx-generic-tl-mr3020-v1-squashfs-factory.bin


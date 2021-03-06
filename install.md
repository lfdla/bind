# Installation note

## Software

### Configure development environment

[Armbian Developer Guide](https://docs.armbian.com/Developer-Guide_Build-Preparation/)

* Install Ubuntu Xenial 16.04 x64 (basic system, OpenSSH and Samba)
* Login as root and run:

```bash
# apt-get -y -qq install git
# git clone --depth 1 https://github.com/armbian/build
# cd build
```

* Run the script:

```
# ./compile.sh
```

![](http://www.armbian.com/wp-content/uploads/2016/01/21.png)

The images will be in `build/output/images/`

* Mount the image locally for editing:

```
# parted build/output/images/Armbian.img
GNU Parted 3.2
Using build/output/images/Armbian.img
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) unit b
(parted) print
Model: (file)
Disk build/output/images/Armbian.img: 6872367104B
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number Start End Size Type File system Flags
 1 1048576B 31457279B 30408704B primary fat16 lba
 2 31457280B 543162367B 511705088B primary ext4 boot
 3 543162368B 1054867455B 511705088B primary linux-swap(v1)
 4 1054867456B 6554648575B 5499781120B primary ext4

(parted) quit

# mount -o loop,offset=1054867456 build/output/images/Armbian.img mnt
```
### Optimise power consumption

* Create a FEX file with the `bin2fex` tool
* Edit the following parameters:

```
[target]
boot_clock = 912

[dram_para]
dram_clk = 408

[hdmi_para]
hdmi_used = 0

[tv_para]
tv_used = 0

[tvout_para]
tvout_used = 0

[wifi_para]
wifi_used = 0

[pcm0]
daudio_used = 0

[audio0]
audio_used = 0

[dvfs_table]
max_freq = 912000000
min_freq = 240000000

[gpio_para]
gpio_used = 1
gpio_num = 2
gpio_pin_1 = port:PA00<0><2><default><default>
gpio_pin_2 = port:PA01<1><0><0><0>
```

* Create the bin with `fex2bin` ant put it back to /boot

### Patch the image for OTG

* Patch kernel modules:

```
echo "g_ether" > mnt/etc/modules
echo "gpio-sunxi" > mnt/etc/modules
```

* Patch kernel module options: `echo "options g_ether use_eem=0" > /etc/modprobe.d/g_ether.conf`
* Configure interface: `vi /etc/network/interfaces.d/usb0`

```
allow-hotplug usb0
iface usb0 inet static
	address 10.42.0.1
	netmask 255.255.255.0
	dns-nameservers 8.8.8.8
	pre-up /bin/sh -c 'echo -n 2 > /sys/bus/platform/devices/sunxi_usb_udc/otg_role'
```

* Configure interface: `vi /etc/network/interfaces.d/eth0`

```
auto eth0
allow-hotplug eth0
iface eth0 inet static
        address 192.168.42.1
        netmask 255.255.255.0
```

`echo "UseDNS no" >> /etc/ssh/sshd_config`

### Install services

```
apt remove avahi-autoipd
systemctl disable ntp
```

#### DHCP

* Update apt: `apt-get update`
* Install DHCP server: `apt-get install isc-dhcp-server`
* Configure `/etc/dhcp/dhcpd.conf`

```
max-lease-time 600;
subnet 10.42.1.0 netmask 255.255.255.0 {
	range 10.42.1.100 10.42.1.200;
}
```

* Enable service: `systemctl enable isc-dhcp-server`

#### Tools

```
apt-get install tcpdump socat netcat udpcast
```


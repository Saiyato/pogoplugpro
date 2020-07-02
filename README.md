# Pogoplug Pro Hack 
The Pogoplug era is long gone, all necessary services have been retired, so it's basically a high-tech brick. Nonetheless, it *is* possible to repurpose this little (not so powerful) device, why, you ask? Well... because we can! I present you the "Pogoplug Pro re-activation project".

Ok, so what are we talking about, and what are the requirements?

1. We need access to the terminal
2. We need to flash a new bootloader
3. We need to create an Ubuntu filesystem to boot from (the NAND only has 128MB)
4. We would like WiFi to be operational
5. We could install something like OMV (OpenMediaVault)

## 1. We need access to the terminal
Back in the days, when you could still boot the little device, you could use a 'secret' command to enable dropbear (and SSH). But, unfortunately, because all services have retired, this will not work anymore (at least, I tried and failed in many occassions).

The command is: `curl -k “https://root:ceadmin@[PogoplugIPAddress]/sqdiag/HBPlug?action=command&command=dropbear%20start”`
You need to change the `[PogoplugIPAddress]` part with the IP-address of your device of course.

So, that's a no go... how do I go about it then? Well you can connect to the board and use the serial connection instead. You will need to open up the device and get your hands on a *JST 2.0mm PH 4-Pin* connector (you can open up some old devices to see if you can find a matching cable, connecting dupont cables wont work, unless you solder them).

###### Connection settings
```
Speed: 115200
Data bits: 8
Stop bits: 1
Parity: None
Flow control: None
```

If you want to enable SSH, you can just edit the */etc/init.d/rcS* file.
First remount the rootfs with read/write using this command `mount -o remount,rw /` and edit `/etc/init.d/rcS` (you must use vi, nano is not yet an option). Comment out `/etc/init.d/hbmgr.sh start` by placing a pound sign before it (like this `#/etc/init.d/hbmgr.sh start`) and add this on a new line `/usr/sbin/dropbear`. This will disable loading the standard Pogoplug software, but instead just load dropbear (and enable SSH).

## 2. We need to flash a new bootloader
Someone else can probably tell you exactly what it does, and what it's for. I will stick to the necessary parts... you need to update it for more freedom and fun.
First things first, let's see if our hardware is Oxsemi, otherwise, just stop right there.
```
#Verify Pogoplug is expected version (Oxnas)
cat /proc/cpuinfo | grep Hardware

#Stop here if not expected output.
#Expected output
#Hardware : Oxsemi NAS

#stop my.pogoplug.com service
killall hbwd
```

Now that's out of the way, we need some tools (see tools tarball in this repo).
`wget https://github.com/Saiyato/pogoplugpro/raw/master/linux-tools-installation-bodhi.tar.gz`

md5:
e58f442411eb35e641d40ea0577e00ff linux-tools-installation-bodhi.tar.gz
sha256:
88dfa8eadb319e2e286320643a654bf89bff0b0d450562fce09938e7f3b0007d linux-tools-installation-bodhi.tar.gz 

Download the tools to */tmp* and make them executable `chmod +x flash_erase fw_printenv fw_setenv nanddump nandwrite`. Setup fw_env.config and backup old envs:
```
#remount '/' as read/write
#by default the Pogoplug OS (internal flash) is read only
mount -o remount,rw /

#setup fw_env.config for oxnas
echo "/dev/mtd0 0x00100000 0x20000 0x20000">/etc/fw_env.config

#save original envs
/usr/local/cloudengines/bin/blparam > /blparam.txt
```

Download the latest uboot tarball and verify hash (uboot.2015.10-tld-2.ox820.bodhi.tar can be found in this repo).

md5sum
c8d52236f906fd7fc8c84d15562331b0
sha256sum
d9e034f70a3c40d6a970084c858e8890989e0d40d11699dc488e3dd852092aad 

```
#extract uBoot files
tar -xf uboot.2013.10-tld-4.ox820.bodhi.tar
```

Now the fun starts, carefully copy the commands, you don't want to brick your device, right?
```
#BE EXTRA CAREFUL WITH THE THESE COMMANDS.
#NO TYPOS! CUT AND PASTE.
#Erase and flash uboot on mtd0
#Flash encoded spl stage1 to 0x0
/tmp/flash_erase /dev/mtd0 0x0 6
/tmp/nandwrite /dev/mtd0 uboot.spl.2013.10.ox820.850mhz.mtd0.img

#Flash uboot to 0x40000
/tmp/nandwrite -s 262144 /dev/mtd0 uboot.2013.10-tld-4.ox820.mtd0.img
#Flash uboot environment
#Erase 1 block starting 0x00100000
/tmp/flash_erase /dev/mtd0 0x00100000 1
/tmp/nandwrite -s 1048576 /dev/mtd0 pogopro_uboot_env.img

#Set MAC Address
/tmp/fw_setenv ethaddr "$(cat /sys/class/net/eth0/address)"
```

Optionally default to Pogoplug classic dtb.
```
#default to pogoplug classic dtb
/tmp/fw_setenv fdt_file '/boot/dts/ox820-pogoplug-classic.dtb'
/tmp/fw_setenv dt_load_dtb 'ext2load usb 0:1 $dtb_addr $fdt_file'
```

Check your MAC address and verify that there are no errors in the uboot env params.
```
#double check the MAC Address matches with
#what is on the bottom of your Pogoplug
/tmp/fw_printenv ethaddr

#print out all uboot environment parameters
#make sure there are no errors
/tmp/fw_printenv > /fw_printenv.txt
/tmp/fw_printenv
```

I've use the serial cable for the entire process, but you can use netconsole I support. See this post: https://forum.doozan.com/read.php?3,14,14

## 3. We need to create an Ubuntu filesystem to boot from (the NAND only has 128MB)


## 4. We would like WiFi to be operational
If */etc/apt/sources.list* does not containt `deb http://ftp.us.debian.org/debian buster main non-free`, add it and run `apt-get update`.

In order to control the WiFi device, we will need some packages. Install them using the following command:
`apt install pciutils wpasupplicant hostapd bridge-utils wireless-tools iw firmware-misc-nonfree firmware-realtek`

#### Check the WiFi device id
Because we can't tell for sure what your device id, you'd best just query it using `iwconfig`. The output should look like this:
```
root@pogoplug:~# iwconfig
wlp0s0    IEEE 802.11  ESSID:off/any
          Mode:Managed  Access Point: Not-Associated   Tx-Power=0 dBm
          Retry short  long limit:2   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:off

sit0      no wireless extensions.

eth0      no wireless extensions.

lo        no wireless extensions.

ip6tnl0   no wireless extensions.
```
In this case my device id is wlp0s0, which I will use in my configs.

#### Add the network config
In order to connect, you will need to setup your wireless network info in */etc/network/interfaces*
```
auto wlp0s0
iface wlp0s0 inet dhcp
  wpa-ssid your_wifi_ssid
  wpa-psk your_wifi_password
```

Test your configuration afterwards with `ifup -v wlp0s0`, you should be seeing some DHCP activity and an assigned IP address.
Note: you will need to add this info again if you've installed OMV afterwards.

#### Enable auto-connect after boot
Luckily someone already did the heavy lifting for this, you can just download the WiFi_Check script from this repo
```
cd /usr/local/bin/
wget --no-check-certificate https://raw.githubusercontent.com/Saiyato/pogoplugpro/master/WiFi_Check
chmod +x WiFi_Check

apt-get install cron

crontab -e
*/3 * * * * /usr/local/bin/WiFi_Check
```

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

md5:
e58f442411eb35e641d40ea0577e00ff linux-tools-installation-bodhi.tar.gz
sha256:
88dfa8eadb319e2e286320643a654bf89bff0b0d450562fce09938e7f3b0007d linux-tools-installation-bodhi.tar.gz 

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

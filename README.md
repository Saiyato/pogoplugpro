# Pogoplug Pro Hack 
The Pogoplug era is long gone, all necessary services have been retired, so it's basically a high-tech brick. Nonetheless, it *is* possible to repurpose this little (not so powerful) device, why, you ask? Well... because we can! I present you the "Pogoplug Pro re-activation project".

Ok, so what are we talking about, and what are the requirements?

1. We need access to the terminal
2. We need to flash a new uBoot
3. We need to create an Ubuntu filesystem to boot from (the NAND only has 128MB)
4. We would like WiFi to be operational
5. We could install something like OMV (OpenMediaVault)

## 1. We need access to the terminal
Back in the days, when you could still boot the little device, you could use a 'secret' command to enable dropbear (and SSH). But, unfortunately, because all services have retired, this will not work anymore (at least, I tried and failed in many occassions).

The command is: `curl -k “https://root:ceadmin@[PogoplugIPAddress]/sqdiag/HBPlug?action=command&command=dropbear%20start”`
You need to change the [PogoplugIPAddress] part with the IP-address of your device of course.

# Freenas / Truenas CUPS server with AirPrint
Quick how-to for setting up a CUPS printing server in Truenas 12 using CUPS with support for AirPrint.

## Jail
Create a new jail called `cups`, setup your network as usual, check the auto-start option and login to it as root. Setup a password so you can add printers later in CUPS, just run as `root`:

`$ passwd`

## Install Packages inside jail

I prefer `vim`, but you can use `nano` if you wish

`$ pkg install vim wget avahi cups python gutenprint-cups cups-filters print/gutenprint py38-pycups nss_mdns`

If you own an HP printer:

`$ pkg install print/hplip`

For an Epson printer like in my case (I own an EcoTank M1120, with wifi but no airprint support)

`$ pkg install epson-inkjet-printer-escpr`

## DEVFS: Allow USB devices inside the jail for CUPS to see the printer

We will accomplish this by using `devfs` rulesets, and mount those devices inside the jail.

First we need to get the currnet `ruleset` for the jail:

`$ iocage get devfs_ruleset cups`

In my case this is ruleset `4`, which seems to be the default for all jails. We can see the ruleset running this in Truenas host as `root`:

`$ devfs rule -s 4 show` 

```
100 include 1
200 include 2
300 include 3
400 path fuse unhide
500 path zfs unhide
```

You can also see rulesets 1, 2 and 3 since those are included in ruleset 4 to see what they do, and you'll see there's no usb devices there.

### Create new devfs ruleset

First let’s check if we see the printer and its name. Run this in the Truenas host as `root`:

`$ usbconfig`

```
usbconfig
ugen1.1: <(0x103c) EHCI root HUB> at usbus1, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=SAVE (0mA)
ugen0.1: <0x8086 XHCI root HUB> at usbus0, cfg=0 md=HOST spd=SUPER (5.0Gbps) pwr=SAVE (0mA)
ugen0.2: <EPSON M1120 Series> at usbus0, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=ON (2mA)
ugen0.3: <American Power Conversion Back-UPS RS 900G FW:879.L4 .I USB FW:L4> at usbus0, cfg=0 md=HOST spd=FULL (12Mbps) pwr=ON (2mA)
ugen0.4: <vendor 0x0424 product 0x2660> at usbus0, cfg=0 md=HOST spd=HIGH (480Mbps) pwr=SAVE (2mA)
ugen0.5: <Norelsys NS1066> at usbus0, cfg=0 md=HOST spd=SUPER (5.0Gbps) pwr=ON (2mA)
```

My EPSON printer is in `ugen0.2` port.

We will create a new ruleset number `1100` that includes ruleset number 4 and unhides the necessary device for the jail. Also USB devices need to be owned by the group cups and permission 660. Normally, the cups port/package adds a file in `/usr/local/etc/devd` to change the group owner and permission when a new USB device is plugged in. However devd doesn’t work in jails, so we will also have to change these using devfs.

On a normal FreeBSD system, we could setup this ruleset in `/etc/devfs.rules`, however this file would be rewritten in FreeNAS on each reboot/update. Instead we create a script that will be started on each boot. We can also use this script to detect the USB port on which the printer is connected. In my case I store this script in my home directory on its own dataset (so it won’t be overwritten on reboot), but you can use any of your datasets. For this example the script will be in `~myuser/cups-devfs-ruleset.sh`.

```
#!/bin/sh
# Custom ruleset for jails

export PATH="/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin"

RULESET=1100
CUPS_GID=193
PRINTER_NAME="EPSON M1120 Series"

# Find the printer device.
UGEN_DEV=$(usbconfig | grep "$PRINTER_NAME" | cut -d':' -f 1)
USB_DEV=$(readlink /dev/$UGEN_DEV)

if [ -z "$UGEN_DEV" -o -z "$USB_DEV" ]
then
  echo "error: cannot find printer '$PRINTER_NAME'"
  echo "error: please check with usbconfig"
  exit 1
fi

echo "Found $PRINTER_NAME on $UGEN_DEV"

# Clean the ruleset
devfs rule -s $RULESET delset

# Include jails default ruleset and unhide USB device.
devfs rule -s $RULESET add include 4
devfs rule -s $RULESET add path usb unhide
devfs rule -s $RULESET add path $USB_DEV mode 660 group $CUPS_GID unhide
devfs rule -s $RULESET add path $UGEN_DEV mode 660 group $CUPS_GID unhide
devfs rule -s $RULESET add path usbctl mode 644 unhide
```

In this script you should edit `PRINTER_NAME` with the name of the printer found with usbconfig. You should also edit `CUPS_GID` with the gid of the cups group inside your jail (generally this is 193) and edit the `RULESET` number if needed.

Now adjust the permission and execute this script. We will also check that the rules are applied correctly:

```
$ chmod a+rx ~myuser/cups-devfs-ruleset.sh
$ sh ~myuser/cups-devfs-ruleset.sh
Found EPSON M1120 Series on ugen0.2
$ devfs rule -s 1100 show
100 include 4
200 path usb unhide
300 path usb/0.2.0 unhide group 193 mode 660
400 path ugen0.2 unhide group 193 mode 660
500 path usbctl unhide mode 644
```

As you can see, the ruleset has been created correctly. We will now change the devfs ruleset used for the jail. In FreeNAS you can either do that in the GUI in `Jails > myjail > EDIT > Jail Properties > devfs_ruleset` or use the command line. Note that the jail has to be stopped before you can do the modifications. Let’s use the command line in our case:

```
$ iocage stop cups
* Stopping cups
  + Executing prestop OK
  + Stopping services OK
  + Tearing down VNET OK
  + Removing devfs_ruleset: 5 OK
  + Removing jail process OK
  + Executing poststop OK
$ iocage set devfs_ruleset=1100 cups
$ iocage start cups
* Starting cups
  + Started OK
  + Using devfs_ruleset: 1100
  + Configuring VNET OK
  + Using IP options: vnet
  + Starting services OK
  + Executing poststart OK
```

Truenas docs also say that you need to enable `allow_mount` with `allow_mount_devfs` in the JAIL.

### Add devfs script as PREINIT script

Go to `Tasks -> Init/Shutdown Scripts` and add our `myuser/cups-devfs-ruleset.sh` as PREINIT script, so it runs on every reboot making the configuration persistent accross reboots.

### Troubleshooting

It seems that this used to work differently in previous Freenas versions, and you might need additional steps to accomplish this to be persistent accross reboots. It wasn't my case using Truenas 12.1, but I'll leave it here in case it doesn't work for you:

Iocage jails have a exec_prestart property executed before applying the devfs ruleset. However if iocage detects that the devfs ruleset does not exist, it will fall back to a default ruleset. In our case, this means that iocage will always revert to this instead of using our ruleset. Instead we could execute the script as an init task in FreeNAS GUI. However it seems that when a jail is onfigured with Auto-Start, it is started before the Pre-Init tasks. Therefore we configure devfs and then start the jail manually. Create another file (for example in ~myuser/start-jails.sh):

```
#!/bin/sh
# Manually start jails.

sh ~myuser/myjail-devfs-ruleset.sh
/usr/local/bin/iocage start myjail
```

Go into `Tasks > Init/Shutdown Scripts` and ADD one with those info:

```
Type: Script
Script: ~myuser/start-jails.sh
When: Post Init
Enabled: Yes
Timeout: 10
```

Another problem is that `iocage` removes the configured devfs ruleset when the jail is stopped. So if you restart after that, the configured ruleset would not exist and iocage would switch to a default empty one that would expose all your devices in the jail. To fix that, we configure a property to also execute our script when the jail stops:

```
$ iocage set exec_poststop=$(ls ~myuser/myjail-devfs-ruleset.sh) myjail
exec_poststop: /usr/bin/true -> ~myuser/myjail-devfs-ruleset.sh
```

Finally go into the FreeNAS GUI and disable Auto-Start (`Jails > myjail > EDIT > Auto-start: off`).

Note that none of these shenanigans would be necessary if iocage did not fall back on another ruleset if the one specifed in the devfs_ruleset property did not exist. But unfortunately, it doesn’t. If you have a nice idea on how we can get around this problem, please feel free to comment below!


## Allow local network access in cupsd.conf 

`$ vim /usr/local/etc/cups/cupsd.conf`

Allow access from local network with `Allow @LOCAL` and also `Listen *:631`:

```
...
Listen *:631
...

# Restrict access to the server...
<Location />
  Order allow,deny
  Allow @LOCAL
</Location>

# Restrict access to the admin pages...
<Location /admin>
  Order allow,deny
  Allow @LOCAL
</Location>
```

## Setup Avahi daemon for airprint

Disable DBUS in avahi:

`$ vim /usr/local/etc/avahi/avahi-daemon.conf`

```
[server]
...
enable-dbus=no
...
```

Create these two files for URF support.

```	
echo "image/urf urf string(0,UNIRAST<00>)" > /usr/local/share/cups/mime/airprint.types
echo "image/urf application/vnd.cups-postscript 66 pdftops" > /usr/local/share/cups/mime/airprint.convs
```

## Start services and add the printer

Now we are finally ready to add the printer in CUPS. First we need to enable and start the services:

`$ vim /etc/rc.conf`

```
cupsd_enable="YES"
devfs_system_ruleset="system"
avahi_daemon_enable="YES"
```

`$ service cupsd start ; service avahi-daemon start`

You can now go to CUPS admin UI and try to add the printer:

`http://[jail-ip-address]:631/admin`

Click on `Add Printer` it will ask for the root user / password. In my case, my Epson printer showed as `Local Printers: 	EPSON M1120 Series (EPSON M1120 Series)` and the driver showed automatically thanks to the `epson-inkjet-printer-escpr` we installed previously.

## Generate AirPrint services

Now, we will use a helper python script to generate needed AirPrint services files for our printer. 

```
$ wget -O airprint-generate.py --no-check-certificate https://raw.githubusercontent.com/tjfontaine/airprint-generate/master/airprint-generate.py 
$ python3.8 airprint-generate.py
$ mv AirPrint-*.service /usr/local/etc/avahi/services
$ chmod 444 /usr/local/etc/avahi/services/AirPrint-*.service
```
Mine looks like this:
```
$ cat /usr/local/etc/avahi/services/AirPrint-EPSON_M1120_Series.service

<?xml version="1.0" ?><!DOCTYPE service-group  SYSTEM 'avahi-service.dtd'><service-group><name replace-wildcards="yes">AirPrint EPSON_M1120_Series @ %h</name><service><type>_ipp._tcp</type><subtype>_universal._sub._ipp._tcp</subtype><port>631</port><txt-record>txtvers=1</txt-record><txt-record>qtotal=1</txt-record><txt-record>Transparent=T</txt-record><txt-record>URF=none</txt-record><txt-record>rp=printers/EPSON_M1120_Series</txt-record><txt-record>note=EPSON M1120 Series</txt-record><txt-record>product=(GPL Ghostscript)</txt-record><txt-record>printer-state=3</txt-record><txt-record>printer-type=0x2100c</txt-record><txt-record>pdl=application/octet-stream,application/pdf,application/postscript,application/vnd.cups-raster,image/gif,image/jpeg,image/png,image/tiff,image/urf,text/html,text/plain,application/vnd.adobe-reader-postscript,application/vnd.cups-pdf</txt-record></service>

```
We also need to disable unneeded ssh service in Avahi:

```
$ cd /usr/local/etc/avahi/services/
$ mv ssh.service ssh.service_bak
$ mv sftp-ssh.service sftp-ssh.service_bak
```

Restart Avahi:

`$ service restart avahi-daemon`

You should now see your printer using AirPrint.

# References
https://hauweele.net/~gawen/blog/?p=2532

https://www.kaisblog.de/2014/08/27/airprint-jail-freenas/

https://docs.freebsd.org/en/articles/cups/#printing-cups-configuring-server
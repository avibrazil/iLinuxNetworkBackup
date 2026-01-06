OK guys, I'm moving to 100% satisfying solution with [libimobiledevice](https://libimobiledevice.org/), which is packaged in most Linux distributions, and [Jackson Coxson's netmuxd](https://github.com/jkcoxson/netmuxd). Netmuxd complements usbmuxd with network capabilities and both must run together.

Netmuxd was not available in my platform package set, and OpenSSL needs a little tweeking, so a preparation script is required. Here are the steps to prepare my Fedora 42/43 and **successfully backup my iPhone over the network**. Nothing has to be done as root, **all scripts and commands executed by my regular user**.

## Prepare things

Required packages:
```shell
dnf install -y libimobiledevice-utils
```

Following script will discover [latest version of netmuxd](https://github.com/jkcoxson/netmuxd/releases/tag/v0.3.0) for my platform and download it, and will prepare a configuration file for OpenSSL, so pairing won't fail.
```shell
#!/bin/sh

folder=/media/Backup/MobileSync
netmuxd=netmuxd-x86_64-linux-gnu

echo "Preparing compatible OpenSSL configuration"

# From  https://github.com/libimobiledevice/libimobiledevice/issues/1606#issuecomment-2543116644
cat <<EOF > "$folder/openssl-weak.conf"
.include /etc/ssl/openssl.cnf
[openssl_init]
alg_section = evp_properties
[evp_properties]
rh-allow-sha1-signatures = yes
EOF

echo "Getting netmuxd binary..."

latest_netmuxd=`wget -q -O - https://api.github.com/repos/jkcoxson/netmuxd/releases/latest | jq -r '.assets[].browser_download_url' | grep $netmuxd`

wget -q $latest_netmuxd -O "$folder/$netmuxd"

chmod a+x "$folder/$netmuxd"
```

## First pair device with a USB cable
We'll use the OpenSSL configuration file created above
```shell
OPENSSL_CONF=/media/Backup/MobileSync/openssl-weak.conf idevicepair pair
```
This creates some required signatures in `/var/lib/lockdown/{device_UDID}.plist`

## Make backup
This method assumes usbmuxd is also running in your system, which is the regular setup for most Linux distros. [Other methods are documented in netmuxd home page](https://github.com/jkcoxson/netmuxd).

This script runs netmuxd, waits a bit until it finds my device in the network and then use `idevicebackup2` to make the backup. A folder named with my device's UDID will be created under `$folder` (if it doesn't exists).

Since backup is mostly controlled by the device, this method also works for incremental backups even if the backup folder already has a backup created by Apple official software.

Your device will ask for your pin and then start backup.
```shell
#!/bin/sh

folder=/media/Backup/MobileSync
netmuxd=netmuxd-x86_64-linux-gnu

"$folder/$netmuxd" --disable-unix --host 127.0.0.1 &
pid=$!

# Wait a little bit until netmuxd finds device
sleep 15

# Start (incremental) backup
USBMUXD_SOCKET_ADDRESS=127.0.0.1:27015 OPENSSL_CONF="$folder/openssl-weak.conf" idevicebackup2 backup -n --full "$folder"

kill $pid
```
You might need to change these scripts a bit if you have multiple devices to backup.

iOS backup over the network takes a long time, like 20 minutes for an incremental backup, but is pretty solid and stable and I can use my device while roaming through multiple WiFi APs at home.


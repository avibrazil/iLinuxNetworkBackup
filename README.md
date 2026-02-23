OK guys, I'm moving to 100% satisfying solution with [libimobiledevice](https://libimobiledevice.org/), which is packaged in most Linux distributions, and [Jackson Coxson's netmuxd](https://github.com/jkcoxson/netmuxd). Netmuxd complements usbmuxd with network capabilities and both must run together.

Netmuxd was not available in my platform package set, and OpenSSL needs a little tweeking, so a preparation script is required. Here are the steps to prepare my Fedora 42/43 and **successfully backup my iPhone over the network**. Nothing has to be done as root, **all scripts and commands executed by my regular user**.

## Prepare things

Required packages:
```shell
dnf install -y libimobiledevice-utils wget jq
```

Edit the `prepare` and `backup` scripts and change path of folder that will
contain backups for your iDevices. If your platform isn‘t x86_64, also change
the binary name of `netmuxd` that will be downloaded. Mine has:

```shell
folder=/media/Backup/MobileSync
netmuxd=netmuxd-x86_64-linux-gnu
```

I’ll also set and environment variable here just to make this documentation
more readable and precise:

```shell
IBACKUP=/media/Backup/MobileSync
```

## Prepare you computer for wireless iBackup

Run the `prepare` script. It will discover [latest version of netmuxd](https://github.com/jkcoxson/netmuxd/releases/tag/v0.3.0)
for my platform and download it, and will prepare a configuration file for
OpenSSL, so pairing won't fail.

```shell
./prepare
```

## Pair device with a USB cable
We'll use the OpenSSL configuration file created by `prepare` script.

```shell
OPENSSL_CONF=$IBACKUP/openssl-weak.conf idevicepair pair
```
This creates some required signatures in `/var/lib/lockdown/{device_UDID}.plist`

Note that I‘m passing the OpenSSL configuration file that was created in the
folder mentioned above. You‘ll have to change this command a bit to match your
installation.

## Make backup over WiFi

```shell
./backup
```

This method assumes `usbmuxd` is also running in your system, which is the regular setup for most Linux distros.
[Other methods are documented in netmuxd home page](https://github.com/jkcoxson/netmuxd).

This script runs `netmuxd`, waits a bit until it finds my device in the network
and then use `idevicebackup2` to make the backup. A folder named with my
device’s UDID will be created under `$folder` (if it doesn’t exists).

Since backup is mostly controlled by the device, this method also works for
incremental backups even if the backup folder already has a backup created by
Apple official software for macOS or Windows.

Your device will ask for your pin and then start backup.

iOS backup over the network takes a long time, like 20 minutes for an
incremental backup, but is pretty solid and stable and I can use my device
while roaming through multiple WiFi APs at home.

I full backup will take even longer. But you can use your device while backup is
happening.

## Handling multiple devices

Script will get confused if it senses multiple iOS/iPadOS devices on local
network. So you can pass a UUID as parameter to my `backup` script to select
the correct device:

```shell
./backup 00012340-000C3D654321001C
```

But thats annoying, so you can give devices some nicknames via a symbolic link.
Like this:

```shell
cd $IBACKUP
ln -s 00012340-000C3D654321001C my
ln -s 00043210-000C3D654321001C wife
ln -s 00011220-000C3D654321001C son
```

Then backup your wife’s device like this:
```shell
./backup wife
```

Backup yours:
```shell
./backup my
```

# Desire

I‘d love to see someone creating a web UI with Open ID Connect login to let all
my family go there and start a backup.
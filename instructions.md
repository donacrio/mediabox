# Raspberry Media box

# OSMC

We will use [OSMC](https://osmc.tv/) media center to create our Media Box and play Movies & TV shows on TV.

## Installation

Go on the official [OSMC download](https://osmc.tv/download/) page and download the installer

Insert your SD card and launch the installer. Follow the instructions. When finished eject your card, and insert it in your Raspberry. Connect it to your TV and boot.

During initial boot, OSMC will be installed. It might take a few minutes. Then use your TV remote (or Yaste app on your smartphone) and follow the configuration steps.

## Configuration

Go to Settings > MyOSMC > App Store and install the “Cron Task Scheduler” and the “Transmission Torrent Client”.

Go to Settings > MyOSMC > Services and check that ssh and transmission services are enabled.

At this point you can ssh to your Raspberry using the default username/password: osmc/osmc.

NB: It is recommended to change the password using `passwd` command.

# ExpressVPN

In order to download our torrents, we need to hide behind a VPN. You can use any solution (OpenVPN, ExpressVPN, etc…).

I’m subscribed to ExpressVPN so I followed [those steps](https://www.expressvpn.com/fr/support/vpn-setup/app-for-linux/#command-line) to download and install the debian package on the Raspberry. I used `scp` command to send the `.deb` file through ssh. I also enable auto connect at startup.

# Hard Drive

# Installation

First we need to create a new partition table on the HDD:

```bash
# We use parted to write the new partition table
$ sudo umount /dev/sda1
$ sudo parted /dev

# In the parted CLI
(parted) mktable gpt # Create a gpt table
(parted) mkpart primary ext4 0GB 4000GB # Create a ext4 primary partition of 4TB
```

Now if we print the table we can see the following output:

```bash
(parted) p
Model: WD easystore 2648 (scsi)
Disk /dev/sda: 4001GB
Sector size (logical/physical): 512B/4096B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  4000GB  4000GB  ext4         primary
```

We can exit with `quit`, the ext4 partition is now written on the disk (here `/dev/sda1`):

```bash
$ sudo fdisk -l
...
Disk /dev/sda: 3.64 TiB, 4000752599040 bytes, 7813969920 sectors
Disk model: easystore 2648
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 4D327C93-6400-482D-95A3-46FE7EC072F8

Device     Start        End    Sectors  Size Type
/dev/sda1   2048 5859375103 5859373056  3.4T Linux filesystem
```

Now let’s create the ext4 filesystem on the HDD:

```bash
$ sudo mkfs.ext4 /dev/sda1
```

Finally we mount the HDD on the Raspberry :

```bash
$ sudo mkdir /mnt/media
$ sudo mount /dev/sda1 /mnt/media
```

Because we need the HDD to be mounted at startup, we need to add the following line to `/etc/fstab`:

```bash
/dev/sda1 /mnt/media ext4 defaults 0 0
```

If we reboot and run `lsblk -f` we can see that the HDD was mounted at startup:

```bash
NAME        FSTYPE FSVER LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sda
└─sda1      ext4   1.0               ba832796-ef8e-423f-b8f4-b77fdd0f70d9    3.4T     0% /mnt/media
```

## Folder structure

Create the following folder structure:

```bash
/mnt/media                       (root)
.........../downloads
....................../series    (downseries)
....................../movies    (downmovies)
.........../series               (series)
.........../movies               (movies)
```

# Transmission

## Installation

Transmission needs to be installed first using MyOSMC (see above steps).

## Configuration

First stop transmission service:

```bash
sudo service transmission stop
```

Then edit `/home/osmc/.config/transmission-daemon/settings.json` with the following content:

[mediabox/settings.json at main · donacrio/mediabox](https://github.com/donacrio/mediabox/blob/main/transmission/settings.json)

# Flexget

# Installation

First we need to install pip and virtualenv to prepare Flexget installation:

```bash
sudo apt-get -y install python3 python3-pip python3-libtorrent g++
sudo pip3 install --upgrade setuptools
sudo pip3 install virtualenv
```

Now install Flexget

```bash
virtualenv --system-site-packages -p python3 /home/osmc/flexget/
cd /home/osmc/flexget/
bin/pip3 install flexget
source ~/flexget/bin/activate

```

We also need a few additional packages for Flexget to work properly:

```bash
pip3 install subliminal>=2.0
pip3 install transmission-rpc
pip3 install transmission-rpc --upgrade
```

Finally, we will configure systemd to launch Flexget at startup. Create `/lib/systemd/system/flexget.service` with the following content:

[mediabox/flexget.service at main · donacrio/mediabox](https://github.com/donacrio/mediabox/blob/main/flexget/flexget.service)

Now run:

```bash
sudo chmod 755 /lib/systemd/system/flexget.service
sudo systemctl enable flexget
```

## Config

I used the following yaml files for configuration:

[mediabox/config.yml at main · donacrio/mediabox](https://github.com/donacrio/mediabox/blob/main/flexget/config.yml)

[mediabox/secrets.yml at main · donacrio/mediabox](https://github.com/donacrio/mediabox/blob/main/flexget/secrets.yml)

You need to create a  [Trakt.tv](http://Trakt.tv) account and create a `series` and `movies` list.

Now run:

```bash
/home/osmc/flexget/bin/flexget trakt auth <username>
```

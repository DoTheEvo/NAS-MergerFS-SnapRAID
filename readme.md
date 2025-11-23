# A NAS with MergerFS and SnapRAID

###### guide-by-example

WORK IN PROGRESS 

WORK IN PROGRESS 

WORK IN PROGRESS

![duf-pic](https://i.imgur.com/HRD6Al1.png)

A **NAS** that offers:

  * **Mixing** HDDs of various sizes.
  * Trivial to **add** more storage at any time.
  * A **parity drive** protects all data drives.
  * Free and **open source**.

But:
  
  * It's **more work** than just spinning up TrueNAS or OMV.<br>
    Though individual steps are simple, overall it's still a lot of work.
  * Requires solid **knowledge** of linux and the terminal.
  * SnapRAID **isn't realtime**.<br>
    Can recover dead disk only to the state from the last snapraid sync run,
    usually every 24h, and only if the data on the other drives did not change much.
  * Bad for **frequently changed** data, or millions of small files.<br>
    But that means ideal for media server - movies, shows, music, photos, audiobooks,...

# Chapters:

* [Linux and disks preparation](#Linux-and-disks-preparation)
* [MergerFS](#MergerFS) - merge many disks in to one mount point
* [SnapRAID](#SnapRAID) - protect against a disk failure
* [Network File Sharing - Samba and NFS](#Network-File-Sharing---Samba-and-NFS)
* [Spinning down the HDDs](#Spinning-down-the-HDDs)
* [Hardware](#Hardware)

# Linux and disks preparation

![linux-distros](https://i.imgur.com/fUUP3VA.png)

Have a Linux installed on a machine.<br>
I use Arch, installing it using [archinstall](https://wiki.archlinux.org/title/Archinstall),
and have [ansible playbooks](https://github.com/DoTheEvo/ansible-arch)
to set it up how I like it.

Format and partition disks and mount them using fstab.

* Create a new partition **table** on each disk.<br>
  `sudo parted /dev/sdb --script mklabel gpt`<br>
  `sudo parted /dev/sdc --script mklabel gpt`<br>
  `sudo parted /dev/sdd --script mklabel gpt`<br>
* **Partition** disks.<br>
  `sudo parted /dev/sdb --script mkpart primary ext4 0% 100%`<br>
  `sudo parted /dev/sdc --script mkpart primary ext4 0% 100%`<br>
  `sudo parted /dev/sdd --script mkpart primary ext4 0% 100%`<br>
* **Format** the partitions and label them.<br>
  `sudo mkfs.ext4 /dev/sdb1 -L disk_1`<br>
  `sudo mkfs.ext4 /dev/sdc1 -L disk_2`<br>
  `sudo mkfs.ext4 /dev/sdd1 -L disk_3`<br>
* Create **directories** where drives will be mounted.<br>
  `sudo mkdir -p /mnt/disk_1 /mnt/disk_2 /mnt/disk_3`
* Edit **fstab** to mount drives on boot.<br>
  To make things easier, here’s **a script that generates the fstab entries**.<br>
  It includes disk sizes and serial numbers,
  as well as a commented out mergerfs section that can be used later.
  <details>
  <summary><h5>disks-fstab-entries.sh</h5></summary>

  ```bash
  #!/usr/bin/env bash
  # Generates a nicely formatted fstab for ext4 partitions and a MergerFS pool
  # Includes size and serial number for reference

  echo "# ======================================================"
  echo "# Individual disks"
  echo "# ======================================================"
  echo

  # Loop over all partitions with ext4
  lsblk -ln -o NAME,UUID,FSTYPE,SIZE,PKNAME | while read name uuid fstype size pkname; do
      if [ "$fstype" = "ext4" ]; then
          serial=$(lsblk -dn -o SERIAL /dev/$pkname 2>/dev/null)
          echo "# $name ($size, SN: ${serial:-unknown})"
          echo "UUID=$uuid /mnt/disk_X $fstype defaults,nofail,errors=remount-ro 0 2"
          echo
      fi
  done

  echo "# ======================================================"
  echo "# MergerFS pool combining the above disks"
  echo "# ======================================================"
  echo
  echo "# /mnt/disk_* /mnt/pool fuse.mergerfs defaults,category.create=pfrd,func.getattr=newest,minfreespace=20G,fsname=mergerfs 0 0"
  ```

  </details>

  * Make the script executable: `chmod +x disks-fstab-entries.sh`
  * Run it: `./disks-fstab-entries.sh`<br>
    It echoes stuff into the terminal for you to copy/paste into fstab.
  * Remove the lines you don't want, **edit the mount points** as needed
  since they all are set to `/mnt/disk_X` 

To mount all disks defined in fstab - `sudo mount -a` or just reboot.<br>
Check if all is fine with `lsblk` and `lsblk -f` and `duf` or `dysk`.

# MergerFS

![fstab-pic](https://i.imgur.com/ouafW81.png)

* [Github](https://github.com/trapexit/mergerfs)
* [The documentation.](https://trapexit.github.io/mergerfs/)

Merges disks of various sizes in to one combined pool.

* When writing to that pool, files are just simply spread across the disks
  and are **plainly accessible**, even if any of the disks would be pulled
  out and placed in another machine.
* Works at the **file level**, as opposed to the block level, and uses **fuse**
  in **user space**, as opposed to be living in the kernel space.
* Think of it as a **virtual mount point**, not a filesystem.
* There is **NO redundancy, NO protection.** MergerFS is all about just
  merging disk space.<br>
  That's why we use SnapRAID later.
* Very **simple setup** and configuration with a single line in `/etc/fstab`
* **Easy to add drives**, even ones already containing data. No rebuild time.
* Written in C and C++.

### MergerFS setup

* **Install** mergerfs.
* Create a directory for the **final mount point**.<br>
  `sudo mkdir /mnt/pool`<br>
* **edit the fstab**, add mergerfs mount definition.<br>
  ```
  /mnt/disk_* /mnt/pool fuse.mergerfs defaults,category.create=pfrd,func.getattr=newest,minfreespace=20G,fsname=mergerfs 0 0
  ```
* 
  <details>
  <summary>*The fstab definition options explained*</summary>

  * `/mnt/disk_*` - is a wildcard that catches all desired disks mounts.<br>
    Could also be explicitly written *"/mnt/disk_1:/mnt/disk_2:/mnt/disk_3"*
  * `/mnt/pool` - where to mount the final combined disk space
  * `fuse.mergerfs` - defines it as a mergerfs fuse union "filesystem"
  * `defaults` - bunch of default fstab mount options
  * `category.create=pfrd` - sets create policy to a random distribution,
    but the available free space affects the odds.
  * `func.getattr=newest` - if situation happens where there are two versions
    of the same file on different disks, this defines which version
    to pick - newest, it's recommended.
  * `minfreespace=20G` - disks that have less than 20GB will not be picked
    to store new files, saves the space for metadata and whatnot.
  * `fsname=mergerfs` - just defines what name to show in info/disk utilities
  * `0 0` - the first zero is some legacy backup dump, and second is about fsck
    on boot. Since mergerfs is union filesystem no fsck for it.

  Looking around the internet there are other mount options being used,
  but digging deeper shown that some were
  [deprecated](https://trapexit.github.io/mergerfs/latest/config/deprecated_options/)
  or were made in to defaults, or were not ideal to pick in the first place.<br>
  Of note is that there is no [caching](https://trapexit.github.io/mergerfs/latest/config/cache/)
  in this setup.

  </details>

* Mount it, or reboot.<br>
  `sudo mount /mnt/pool`
* **Take ownership** of the new mount `sudo chown -R $USER:$USER /mnt/pool`<br>
  The underlying disks should stay owned by root.
* **Done**.

<details>
<summary><h3>MergerFS policies details</h3></summary>

* [The official docs](https://trapexit.github.io/mergerfs/latest/config/functions_categories_policies/)

A major aspect of mergerfs is picking the policy that decides **how the data
are spread** across the drives. 
Reading the official documentation is a **must do** if planning to move away
from the defaults... but to give some quick idea.

Two types

  * **Path Preserving** - The top directory anchors everything to a specific
    drive. Side effect is that if a disk gets full the new write in to
    that directory fails. You you can also use one of the less strict
    path preserving polices, the ones starting with `msp...` that will
    start putting stuff to another drive instead of failing.
  * **Not Path Preserving** - Data go to a disk with the most free space,
    or with the least used space, or disk is picked randomly,
    or some variation. The directories structure plays no role.

The **default** policy is `pfrd` which picks disk randomly,
but the available free space affects the odds.

#### Possibility to mix policies

MergerFS offers fine control and one can have different policy
for directories and for files. 

```
/mnt/disk_* /mnt/pool fuse.mergerfs defaults,func.create=msppfrd,func.mkdir=pfrd,func.getattr=newest,minfreespace=20G,fsname=mergerfs 0 0
```

* `func.create=msppfrd` - path preserving for files
* `func.mkdir=pfrd` - directories are spread randomly

Kinda like the idea of having stuff grouped by a directory, but not all the way
to the top, just the parent directory. Meaning that files that are together
in the same directory end up on the same disk, but directories themselves
are spread around.<br>
Feels like this makes the best use of the aspect of mergerfs where you
get to keep the data on surviving drives after a failure.<br>
But it kinda depends on the type of data you have.

  * Shows, music, audiobooks,... benefit from this approach as just
    having some of the files of a season, or audiobook or an album is the same
    as not having them at all.
  * Photos, documents,.. here this approach might be worse, as a disk failure
    loses you entire year of photos or an entire vacation because photos were
    in a directory titled *"2019"* and *"Vacation Iceland"*. Which might
    be worse than having at least some of them because they were
    spread across drives.

</details>

# SnapRAID

* [Github](https://github.com/amadvance/snapraid)
* [Changelog](https://github.com/amadvance/snapraid/blob/master/HISTORY)
* [The documentation.](https://www.snapraid.it/manual)
* [Arch Wiki](https://wiki.archlinux.org/title/SnapRAID)

Provides parity protection against disk failure and bitrot.<br> 
Works at file level, not disk or block level. Unlike regular raid that is
constant ever present, snapraid needs to have scheduled syncs. Usually once every 24 hours.<br>
In case of a disk failure, you are able to recover data from the last sync run,
but only if the data on the other drives didn't change much, as the parity is
calculated against the content of all data drives.<br>
This behavior makes it good only for data that do not change often,
like movies, shows, music, photos, audiobooks, videos,...<br>
Of note is that snapraid is saving parity information in to a single file - `snapraid.parity`

### SnapRAID setup

* install snapraid.
* Create `/etc/snapraid.conf`
  ```bash
  # Parity file on a dedicated parity disk
  parity /mnt/parity/snapraid.parity

  # Data disks, the order matters!
  disk disk_1 /mnt/disk_1
  disk disk_2 /mnt/disk_2
  disk disk_3 /mnt/disk_3

  # Content file with metadata for recovery
  content /var/lib/snapraid.content
  content /mnt/disk_1/snapraid.content
  content /mnt/disk_2/snapraid.content
  content /mnt/disk_3/snapraid.content

  # Excludes
  exclude /lost+found/
  exclude .Trash-*/
  exclude .recycle/

  # autosave every 50GB
  autosave 50
  ```
* `sudo snapraid sync` - the first initial sync

### SnapRAID Automation and notifications

A script that runs daily and will execute sync, scrub, smartctl
and do some checks and sends ntfy push notifications.<br>
It should be pretty readable to see what's going on.

* create a file `/opt/snapraid-sync-and-maintenance.sh`<br>

  <details>
  <summary><h5>snapraid-sync-and-maintenance.sh</h5></summary>

  ```bash
  #!/bin/bash
  set -euo pipefail   # strict mode for bash

  # -------------------------------------------------------------
  # Configuration
  # -------------------------------------------------------------
  NTFY_TOPIC="https://ntfy.example.com/NAS" 
  MERGERFS_MOUNT="/mnt/pool"

  # -------------------------------------------------------------
  # A function that Sends ntfy notification and logs to journal
  # -------------------------------------------------------------
  notify() {
      local msg="$1"
      printf "%b" "$msg" | /usr/bin/curl -s -d @- "$NTFY_TOPIC" || true
      echo -e "$msg" # To also log to systemd journal
  }

  # -------------------------------------------------------------
  echo "=== SnapRAID maintenance script started $(date +"%F %T") ==="
  
  # -------------------------------------------------------------
  # Check if mergerfs mount point exists
  # -------------------------------------------------------------
  if ! mountpoint -q "$MERGERFS_MOUNT"; then
      notify "❌ mergerfs mount $MERGERFS_MOUNT not accessible!"
      exit 1
  fi

  # -------------------------------------------------------------
  # Check if disks defined in SnapRAID config are mounted
  # -------------------------------------------------------------
  SNAPRAID_DATA_DISKS=$(grep '^disk' /etc/snapraid.conf | awk '{print $2}' || true)

  if [[ -z "$SNAPRAID_DATA_DISKS" ]]; then
      notify "❌ No disks found in SnapRAID config!"
      exit 1
  fi

  for disk in $SNAPRAID_DATA_DISKS; do
      if ! mountpoint -q "$disk"; then
          notify "❌ SnapRAID disk not mounted: $disk!"
          exit 1
      fi
  done

  # -------------------------------------------------------------
  # Do SnapRAID sync
  # -------------------------------------------------------------
  if ! sync_output=$(snapraid sync 2>&1); then
      notify "❌ SnapRAID sync failed:\n$sync_output"
      exit 1
  fi

  # -------------------------------------------------------------
  # Do partial scrub, 15% of data once a month on the 1st
  # -------------------------------------------------------------
  if [[ $(date +%d) -eq 01 ]]; then
      if ! scrub_output=$(snapraid scrub -p 15 2>&1); then
          notify "❌ SnapRAID scrub failed:\n$scrub_output"
      fi
  fi

  # -------------------------------------------------------------
  # SMART health check once a week on sunday
  # -------------------------------------------------------------
  if [[ $(date +%u) -eq 7 ]]; then
      SMART_DISKS=$(smartctl --scan | awk '{print $1}')
      for disk in $SMART_DISKS; do
          echo "$(date +"%F %T") SMART check for $disk"
          if ! smartctl -H "$disk" > /dev/null 2>&1; then
              notify "❌ SMART health check failed for $disk"
          fi
      done
  fi

  # -------------------------------------------------------------
  # Send success ntfy notification
  # -------------------------------------------------------------
  notify "✅ SnapRAID sync and maintenance completed successfully $(date +"%F %T")"
  ```

  </details>

* Make it executable `sudo chmod +x /opt/snapraid-sync-and-maintenance.sh`
* create a systemd unit and a timer files.

  `/etc/systemd/system/snapraid-sync-and-maintenance.service`
  ```bash
  [Unit]
  Description=SnapRAID sync, scrub, smart-check

  [Service]
  Type=oneshot
  ExecStart=/opt/snapraid-sync-and-maintenance.sh
  ```

  `/etc/systemd/system/snapraid-sync-and-maintenance.timer`
  ```bash
  [Unit]
  Description=SnapRAID sync, scrub, smart-check

  [Timer]
  OnCalendar=*-*-* 01:30:00
  Persistent=true

  [Install]
  WantedBy=timers.target
  ```

* enable the timer<br>
  `sudo systemctl enable --now snapraid-sync-and-maintenance.timer`

<details>
<summary><h1>Network File Sharing - Samba and NFS</h1></summary>

* **Samba** - Well supported by all systems - windows, linux, android, macos,...<br>
  also called by the protocol name - SMB or CIFS
* **NFS** - Simple, ideal for sharing files between linux machine.
* **iSCSI** - Sharing network storage as a block device, literally appears
  as an unformatted disk on a client machine. Good support, great performance
  and can be very useful, but impossible to do with file based MergerFS approach.

<details>
<summary><h2>Samba Setup</h2></summary>

[Arch Wiki](https://wiki.archlinux.org/title/Samba)

* install samba
* create a local linux user that will be used to access the share<br>
  `sudo useradd -M -s "$(which nologin)" bastard`
  * `-M` - no home directory
  * `-s /usr/bin/nologin` - no shell access, exact path differs by distro,
    so using $(which nologin) to get it
* add the user to samba and set password<br>
  `sudo smbpasswd -a bastard`
* copy the config below in to `/etc/samba/smb.conf`
* enable smb.service - `sudo systemctl enable --now smb.service`
* I don't install `nmb.service` for the old netbios discovery,
  it's dead technology.
* If windows machines should have the PC appear in network on it's own, 
  install **wsdd** and enable  the service `sudo systemctl enable --now wsdd.service`

`/etc/samba/smb.conf`
```
[global]
   # Security
   security = user
   map to guest = Never
   server min protocol = SMB2

   # Network
   disable netbios = yes
   smb ports = 445
   dns proxy = no
   deadtime = 0

   # MergerFS/FUSE compatibility
   aio read size = 0
   aio write size = 0
   kernel oplocks = no
   posix locking = no
   strict locking = no
   use sendfile = yes
   #use sendfile = no          if transfer issues 
   min receivefile size = 16384
   #min receivefile size = 0   if transfer issues

   # Performance
   socket options = TCP_NODELAY

   # Logging
   log file = /var/log/samba/%m.log
   max log size = 1000

   # Disable printing
   load printers = no

[Pool]
   path = /mnt/pool
   browseable = yes
   writable = yes
   guest ok = no
   valid users = bastard
   create mask = 0664
   directory mask = 0775
   force user = bastard
   force group = bastard

```

### Linux Client

* install samba/cif support if required<br>
  on arch it's `cifs-utils`
* visit the IP of the NAS in your file explorer `smb://10.0.19.80` or `\\10.0.19.80`

#### Permanent Samba mount at boot

Using [systemd mount and automount](https://wiki.archlinux.org/title/Systemd#systemd.mount_-_mounting).<br>
The name of these service files must be a path of the mount location,
using dashes `-` instead of slashes `/`

* Have `/mnt/pool` directory ready on the client - `sudo mkdir /mnt/pool`

`/etc/systemd/system/mnt-pool.mount`
```
[Unit]
Description=Mount MergerFS Pool
After=network-online.target
Wants=network-online.target

[Mount]
What=//10.0.19.80/pool
Where=/mnt/pool
Type=cifs
Options=rw,username=bastard,password=aaa,uid=1000,gid=1000,file_mode=0664,dir_mode=0775,vers=3.0

[Install]
WantedBy=multi-user.target

```

`/etc/systemd/system/mnt-pool.automount`
```
[Unit]
Description=Automount MergerFS Pool
Requires=network-online.target
After=network-online.target

[Automount]
Where=/mnt/pool
TimeoutIdleSec=0

[Install]
WantedBy=multi-user.target
```

Enable the automount service: `sudo systemctl enable --now mnt-pool.automount`

### Windows Client

Should be trivial to access the share over the network, or to map it to a letter.

---
---

</details>

<details>
<summary><h2>NFS Setup</h2></summary>

[Arch Wiki](https://wiki.archlinux.org/title/NFS)

**Ideal** for sharing between **linux machines**, hypervisor,
and also works ok in the apple ecosystem.
Windows has some support, but samba is more reliable for regular use.<br>
**NFS v3** is picked, as oppose to v4, for simplicity, robustness, and performance.<br>
With v3 the access control to shares is only **IP based**,
allowing in specific IPs, or entire networks.

* install nfs<br>
  on arch it's `nfs-utils`
* edit `/etc/nfs.conf` and enable nfs version 3<br>
  ```
  [nfsd]
  vers3 = y
  vers4 = n
  ```
* edit `/etc/exports` adding line that sets up a share<br>
  `/mnt/pool 10.0.19.0/24(rw,no_root_squash,fsid=1)`<br>
  <details>
  <summary>*nfs export options explained*</summary>
  
  * `rw` - read and **write** allowed
  * `no_root_squash` - if mounted at linux that requires to write
    to the share as root, its respected and the files are owned by root, uid 0
  * `fsid=1` - manually set filesystem id, with mergerfs it might be useful
    to have stability
  * `async` - is now default so skipped, better performance on write,
    but on power loss or crashes data might be lost
  * `no_subtree_check` - is now default so skipped, improves reliability
    and performance at the expense of some potential security issues
  * `all_squash` - anyone and everyone who writes, that write is squashed 
    to be done by the anonymous user with specific uid/guid
  * `anonuid=1000,anongid=1000` - defines uid/guid of the anonymous user

  </details>

* enable nfs-server service - `sudo systemctl enable --now nfs-server`
* for file permissions:
  * on the server have your user have ownership of the share
    `sudo chown -R $USER:$USER /mnt/pool`
  * command `id` tells uid/gid of that user of yours.<br>
    if the client side user has the same uid/gid stuff will just work
  * if more separate users on the client side, then create those users
    as local on the server with correct uid/gid and permissions in to the share

### Linux Client

* install nfs<br>
  on arch it's `nfs-utils`
* check if the client sees the export available at the server's ip<br>
  `showmount -e 10.0.19.80`
* one time mount: `sudo mount -t nfs -o vers=3 10.0.19.80:/mnt/pool /mnt/pool`

#### Permanent NFS mount at boot

Using [systemd mount and automount](https://wiki.archlinux.org/title/Systemd#systemd.mount_-_mounting).<br>
The name of these service files must be a path of the mount location,
using dashes `-` instead of slashes `/`

* Have `/mnt/pool` directory ready on the client - `sudo mkdir /mnt/pool`

`/etc/systemd/system/mnt-pool.mount`
```
[Unit]
Description=Mount MergerFS Pool
After=network-online.target
Wants=network-online.target

[Mount]
What=10.0.19.80:/mnt/pool
Where=/mnt/pool
Type=nfs
Options=vers=3

[Install]
WantedBy=multi-user.target

```

`/etc/systemd/system/mnt-pool.automount`
```
[Unit]
Description=Automount MergerFS Pool
Requires=network-online.target
After=network-online.target

[Automount]
Where=/mnt/pool
TimeoutIdleSec=0

[Install]
WantedBy=multi-user.target
```

Enable the automount service: `sudo systemctl enable --now mnt-pool.automount`

### Windows Client

For Windows NFS clients, write access often requires `all_squash`
with a defined anonuid/anongid, because Windows does not send linux uid/gid
information. Something like this:<br>
`/mnt/pool 10.0.19.0/24(rw,async,no_subtree_check,all_squash,anonuid=1000,anongid=1000,fsid=1)`

**Windows Client**

* must be windows pro, not windows home
* add windows component in the control panel - `Services for NFS` - `Client for NFS`
* cmd, not powershell - `mount \\10.0.19.80\mnt\pool N:`

---
---

</details>

</details>

---
---


<details>
<summary><h1>Spinning down the HDDs</h1></summary>

![spindown-gif](https://i.imgur.com/QrhQWQc.gif)

Saves \~3W of power per disk, heat, wear and noise,... but the first time
accessing the storage takes \~10 seconds..<br>
Disks might or might not spindown on their own on idle.
Most distros don't force it, but the firmware preset of the disks might.

If you want to **take control** [hdparm](https://linux.die.net/man/8/hdparm)
can change power saving and idle time.

* `sudo hdparm -B ...` -  controls APM power saving in drives firmware
* `sudo hdparm -S ...` - controls idle standby timer in drives firmware

The execution does not survive reboot so systemd service is used to
apply prefered behaviour on boot.

### Prevent spindown 

`/etc/systemd/system/hdds-spindown-prevent.service`
```bash
[Unit]
Description=Prevent spindown for all HDDs
After=local-fs.target systemd-udev-settle.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for d in /sys/block/sd*; do [ -f "$d/queue/rotational" ] && [ "$(cat $d/queue/rotational)" -eq 1 ] && [ -b "/dev/$(basename $d)" ] && /usr/bin/hdparm -S 0 -B 255 /dev/$(basename $d) 2>/dev/null || true; done'

[Install]
WantedBy=multi-user.target
```

Enable the service<br>
`sudo systemctl enable --now hdds-spindown-prevent.service`

### Control spindown 

`/etc/systemd/system/hdds-spindown-enabled.service`
```bash
[Unit]
Description=Enable spindown for all HDDs after 2 hours idle
After=local-fs.target systemd-udev-settle.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for d in /sys/block/sd*; do [ -f "$d/queue/rotational" ] && [ "$(cat $d/queue/rotational)" -eq 1 ] && [ -b "/dev/$(basename $d)" ] && /usr/bin/hdparm -S 244 -B 127 /dev/$(basename $d) 2>/dev/null || true; done'

[Install]
WantedBy=multi-user.target
```

Enable the service<br>
`sudo systemctl enable --now hdds-spindown-enabled.service`

The 2 hours is set by `hdparm -S 244`,
hdparm uses bit weird [time system](https://linux.die.net/man/8/hdparm).

  * 1 - 240 is multiplies of 5 seconds. So 60 is 5 minutes.
  * 241 - 251 is multiplies of 30 minutes. So 241=30m, 242=60m, 243=90m,...

As for the `hdparm -B 127` - the APM range is 1 - 254.
Lower the number more aggresive power savings.
The 127 value is a moderate balanced level that will not be more aggresive
than the desired 2 hours.

---

Be aware, freshly formated ext4 disks finish their initialization
in the background. Kernel runs `ext4lazyinit` process that zeroes inodes
and disks can be heard chirping and doing something even when there should be
no activity. Though also disks firmware might be doing stuff on its own occasionally.

* A command showing the spindown state of every drive:<br>
  `for d in /dev/sd[a-z]; do echo -n "$d: "; sudo hdparm -C "$d" | grep state; done`
* See disks activity:<br>
  `sudo iotop -ao`<br>
  `sudo blktrace -d /dev/sdd -o - | blkparse -i -`

### Monitoring spindowns and spinups

To have degree of certanty that disks are not spinning down and up
400 times a day, hastening their demise.

* Create the log script in `/opt/disks-spin-logger.sh`<br>
  <details>
  <summary><h5>disks-spin-logger.sh</h5></summary>
  ```bash
  #!/bin/bash
  LOGFILE="/var/log/disks-spin.log"
  STATEFILE="/var/lib/disks-spin.state"
  SUMMARY="/var/log/disks-spin-summary.log"
  DATE=$(date +"%Y-%m-%d %H:%M:%S")
  DAY=$(date +"%Y-%m-%d")

  mkdir -p /var/lib

  for dev in /dev/sd[a-z]; do
      CURR_STATE=$(hdparm -C "$dev" 2>/dev/null | awk '/drive state/ {print $NF}')
      [[ -z "$CURR_STATE" ]] && continue  # skip if no result

      PREV_STATE=$(awk -v d="$dev" '$1==d {print $2}' "$STATEFILE" 2>/dev/null)

      if [[ "$CURR_STATE" != "$PREV_STATE" ]]; then
          echo "$DATE $dev $CURR_STATE" >> "$LOGFILE"

          # count spinups / spindowns
          case "$CURR_STATE" in
              active*|idle*)  # matches "active", "active/idle", "idle"
                  EVENT="SPINUP"
                  ;;
              standby)
                  EVENT="SPINDOWN"
                  ;;
              *)
                  EVENT=""
                  ;;
          esac

          if [[ -n "$EVENT" ]]; then
              # Update summary: device + day + counters
              awk -v dev="$dev" -v day="$DAY" -v event="$EVENT" '
                  BEGIN {found=0}
                  {
                      if ($1==day && $2==dev) {
                          if (event=="SPINUP")   $3++
                          if (event=="SPINDOWN") $4++
                          found=1
                      }
                      print
                  }
                  END {
                      if (!found) {
                          up=(event=="SPINUP")?1:0
                          down=(event=="SPINDOWN")?1:0
                          print day, dev, up, down
                      }
                  }
              ' "$SUMMARY" 2>/dev/null > "$SUMMARY.tmp"
              mv "$SUMMARY.tmp" "$SUMMARY"

              if ! grep -q "^DATE" "$SUMMARY"; then
                  sed -i '1iDATE DEVICE SPINUPS SPINDOWNS' "$SUMMARY"
              fi

          fi

          # update state file
          grep -v "^$dev " "$STATEFILE" 2>/dev/null > "$STATEFILE.tmp"
          echo "$dev $CURR_STATE" >> "$STATEFILE.tmp"
          mv "$STATEFILE.tmp" "$STATEFILE"
      fi
  done
  ```
* Make the script executable: `sudo chmod +x /opt/disks-spin-logger.sh`
* create a systemd unit file and a timer file in `/etc/systemd/system/`.

  `disks-spin-logger.service`
  ```bash
  [Unit]
  Description=Log disks spin state

  [Service]
  Type=oneshot
  ExecStart=/opt/disks-spin-logger.sh
  ```

  `disks-spin-logger.timer`
  ```bash
  [Unit]
  Description=Run disks spin logger periodically

  [Timer]
  OnBootSec=1min
  OnUnitActiveSec=5min
  AccuracySec=10s
  Persistent=true

  [Install]
  WantedBy=timers.target
  ```

* enable the timer<br>
  `sudo systemctl enable --now disks-spin-logger.timer`

The log files will be in `/var/log/`

  * `disks-spin.log` - state of every disk at the check interval.
  * `disks-spin-summary.log` - total number of spin downs and ups per day

#### logrotate

To prevent growth of the log files.

* install `logrotate`
* create a config file<br>
  `/etc/logrotate.d/disks-spin`
  ```bash
  /var/log/disks-spin*.log {
      size 20M
      rotate 1
      missingok
      notifempty
      create
  }
  ```
Can do dry run to see what it would do `sudo logrotate -d /etc/logrotate.conf`
</details>

---
---

<details>
<summary><h1>Hardware</h1></summary>

![pic-nas-cases](https://i.imgur.com/dnUzBYJ.jpeg)

### Case

How big or small, how expensive, how many 3.5"/2.5"/m.2 disks positions?<br>
My pick - mATX case from aliexpres - [Sagittarius](https://youtu.be/fjqKEmNot_M)
  
  * was 150€ with shippin
  * I like the idea of a smaller case, but not too small where
    ITX motherboards are expensive and much more limiting with PCIE slots.
    Same with SFX power supplies as they got much more expensive.
  * 8x 3.5" hot swappable disk bays, sata / sas compatible
  * great cooling with space for 2x 120fans just for disks,
    and another 2x 120 for the mobo section

Other popular cases

  * Jonsbo N line has nice ITX cases, but they kinda dropped the ball
    with mATX cases.
  * Fractal design is often picked, with node line for smaller cases,
  and Define R5 R6 for when big case is not an issues and you want lot of positions.
  * InWin Chopin MAX - when no 3.5" disks are needed and just want a tiny case.

---

![hba-cards-pic](https://i.imgur.com/BtcqZSB.jpeg)

### HBA card

If you have more than 4x disks, it might be difficult finding a motherboard
with enough sata ports. Or if you plan to to run a hypervisor on metal
and as a VM run TrueNAS, you need an HBA card that you passthrough in to
the VM so that TrueNAS has full access to disks without any abstraction layers.

The ideal solution is to buy a used enterprise-tier raid card
in IT mode.<br>
IT mode means raid functionality is disabled and it's just a pcie card that
provides you with plenty of sata connection for the drives.
But it is of high quality as you don't want to be trying
to solve - *why some disks are disconnecting sometimes*, or whatever issues...<br>
These cards cost like 400€ new, but you can get them cheap on ebay.

General cards overview - [youtube video](https://youtu.be/hTbKzQZk21w)

**The naming scheme**

* LSI - the company that manufactures the cards, owned by Broadcom
* SAS2008; SAS3008; SAS3408; SAS3808 - specific chip on the card
* 9211-8i; 9300-8i; 9400-8i; 9500-8i - specific model of the card,
  usually with x8 disks connections
* various cable types connector - SFF-8087; SFF-8643; SFF-8654;...

For just HDDs 9211-8i used to be the go-to recommendation
since they were cheap and the performance is more than enough
for spinning drives.<br>
But then the price of 9300-8i started to drop and is about 40€,
So why not get something newer...

But then theres power consumption.<br>
[A video](https://www.youtube.com/watch?v=tQOI2pUUVDE)
came out measuring power consumption of these, and the 9300 was idling
at highest 11W, the newer and more expensive 9400 was 7W and 9500 was 6W.
The old 9211 was 7W. So take that in to consideration.

And to repeat, it must be in the IT mode and ideally ordered straight away
with cables. There are sellers with 99%+ positive feedback with 150k+ items
sold, so one should not feel scared of buying these.
I tend to go for Fujitsu cards.

---

![cpus-mobo-pic](https://i.imgur.com/Y7CGiFF.jpeg)

### CPU, motherboard, ram

Can be anything you have on hand, can be something really special,
all depending on needs, wants and the budget.

Big decision Intel vs AMD.

* Intel - bit [better igpu performance](https://github.com/DoTheEvo/selfhosted-apps-docker/tree/master/jellyfin#testing-various-hardware)
  for video transcoding, if planing to run something like jellyfin...
* AMD - possibility to go ECC ram with some CPUs, which improves stability
  and reliability a bit

You can also go budget option with something like N100 based motherboard,
like [ASRock N100M](https://www.asrock.com/mb/Intel/N100M/),
but not having 2.5gbit network card is kinda deal breaker for me.

Currently in my NAS I have old pentium G3240, waiting for me to decide what
I like to to put in.

---

![psu-pic](https://i.imgur.com/8Aiosug.jpeg)

### Power supply

[PSU tier list](https://www.reddit.com/r/pcmasterrace/comments/1iry3a3/new_and_updated_psu_tier_list_is_out/)

My go-to used to be seasonic, but they started to get expensive with PSUs
actually manufactured by them.

Last few years I went with ADATA XPG core reactor II (not the VE version).
It's manufactured by CWT, has A+ rating on the tier list, 10 years warranty
and cost me 80€.<br> 
Switching from a very old seasonic that my NAS had, power consumption went
from 26W idle to 22W idle. Fucking stiff cables though.

---

![network-stuff-pic](https://i.imgur.com/gQ8hIrH.jpeg)

### Network cards

Plan ahead, higher speed NICs require also higher speed switches.

**2.5gbit**

Started to be really affordable. There are switches from ubiquiti and mikrotik
and 2.5gbit NICs are really common in new motherboards now.
So it's not that expensive and usually worth it, as even a single HDD
is 2x faster than the pathetic 1gbit network speed.

**10gbit**

People like the idea and the cost of getting there is doable,
but if in planning stage and your situation allows it...
plan **not to** go for 10GBase-T, meaning not going for copper twister pair rj45
cables and buying 10gbit rj45 switches and NICs. They tend to run very hot,
with high power consumption. But I understand, if you already have cat6a
cable run... it's bothersome to start to think about cables.<br>
But if your situation allows, consider going for SFP+ switches and NICs.
For distances under 7m you use cheap and very reliable DAC cables.
For longer runs its optical with transceivers.<br>
SFP+ switches run cool enough to be passively cooled.
Popular cheap choices are Mikrotik CRS305-1G-4S+IN and CRS309-1G-8S+.
But many of their switches have few SFP+ ports.<br>
For 10gbit network cards popular choice is used ebay - intel x520s or x710,
or Mellanox ConnectX-3, ConnectX-4, ConnectX-5.<br>
Older cheaper ones, might not support ASPM, which means they will not let
CPU reach higher states of power saving. X710 has good reputation on that,
but then it also has reputation that it is very picky about cables and transceivers...

Some discussion [here,](https://forums.servethehome.com/index.php?threads/sfp-cards-with-aspm-support.36817/page-4)
some guide [here](https://www.reddit.com/r/homelab/comments/1jddpus/mellanox_nic_firmwareconfiguration_guide/)
</details>

---
---

# Other Guides

* https://perfectmediaserver.com/02-tech-stack/mergerfs/
* https://perfectmediaserver.com/03-installation/manual-install-ubuntu/
* https://thenomadcode.tech/mergerfs-snapraid-is-the-new-raid-5
* https://zackreed.me/snapraid-split-parity-sync-script/
* https://www.youtube.com/watch?v=Yt67zz9p0FU
* https://www.reddit.com/r/OpenMediaVault/comments/11gwi1g/significant_samba_speedperformance_improvement_by/

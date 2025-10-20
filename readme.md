# NAS with MergerFS and SnapRAID

###### guide-by-example

WORK IN PROGRESS WORK IN PROGRESS WORK IN PROGRESS

![duf-pic](https://i.imgur.com/HRD6Al1.png)

A **NAS** that offers:

  * **Mixing** HDDs of various sizes.
  * Trivial to **add** more storage at any time.
  * One **parity drive** protects all data drives.
  * Free and **open source**.

But:
  
  * It's **more work** than just spinnig TrueNAS or OMV.<br>
    Though steps alone are simple, straightforward, but overal it's still a lot.
  * Requires solid **knowledge** of linux and the terminal.
  * SnapRAID **isn't realtime**.<br>
    Can recover dead disk only to the state from the last snapraid sync run,
    usually every 24h, and only if the data on the other drives did not change much.
  * Bad for **freqently changed** data, or millions of small files.<br>
    But that means ideal for media - movies, shows, music, photos, audiobooks,...

### Chapters:

* Linux and disks preparation
* MergerFS - merge disks in to one mount point
* SnapRAID - adding a parity drive to prevent data loss on a disk failure
* Sharing the storage - SMB, NFS, iSCSI
* Spinning down the HDDs
* Hardware

# Linux and disks preparation

![linux-distros](https://i.imgur.com/fUUP3VA.png)

Have a linux installed on a machine.<br>
I use Arch, installing it using archinstall,
and have [an ansible playbooks](https://github.com/DoTheEvo/ansible-arch)
to set it up how I like it.

Now format and partition disks and mount them using fstab.

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
  To make things easier, here’s a script that **generates the fstab entries**.
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

  It includes the size and the serial number of the disks,
  as well as a commented out mergerfs section that can be used later.
  * Make the script executable: `chmod +x disks-fstab-entries.sh`
  * Run it: `./disks-fstab-entries.sh`<br>
    It echoes stuff into the terminal for you to copy/paste into fstab.
  * Remove the lines you dont want, **edit the mount points** as needed
  since they all are set to `/mnt/disk_X` 

To mount all disks definied in fstab - `sudo mount -a` or just reboot.<br>
Check if all is fine with `lsblk` and `lsblk -f` and `duf` or `dysk`.

# MergerFS

![fstab-pic](https://i.imgur.com/bKhC1zT.png)

* [Github](https://github.com/trapexit/mergerfs)
* [The documentation.](https://trapexit.github.io/mergerfs/)

Merges disks of various sizes in to one combined pool.

* When writing to that pool, files are just simply spread across the disks
  and are **plainly accessible**, even if any of the disks would be pulled
  out and placed in another machine.
* Works at the **file level**, as oppose to the block level, and uses **fuse**
  in **user space**, as oppose to be living in the kernel space.
* Think of it as a **virtual mount point**, not a filesystem.
* There is **NO redundancy, NO protection.** MergerFS is all about just
  merging disk space.<br>
  That's why we use SnapRAID later.
* Very **simple setup** and configuration with a single line in `/etc/fstab`
* **Easy to add drives**, even ones already containing data. No rebuild time.
* Written in C and C++.

### MergerFS setup

All your disks are formated, mounted and ready.

* **Install** mergerfs.
* Create a directory for the **final mount point**.<br>
  `sudo mkdir /mnt/pool`<br>
* **edit the fstab**, add mergerfs mount definition.<br>
  ```
  /mnt/disk_* /mnt/pool fuse.mergerfs defaults,category.create=pfrd,func.getattr=newest,minfreespace=20G,fsname=mergerfs 0 0
  ```
* Mount it, or reboot.<br>
  `sudo mount /mnt/pool`
* **Take ownership** of the new mount `sudo chown $USER:$USER /mnt/pool`<br>
  The underlying disks should stay owned by root.
* **Done**.

<details>
<summary>**used fstab mount options explained**</summary>

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
or were made in to defaults, or were not wise to pick in the first place.<br>
Of note is that there is no [caching](https://trapexit.github.io/mergerfs/latest/config/cache/)
in this setup.

</details>

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
/mnt/disk_* /mnt/pool fuse.mergerfs defaults,func.create=eppfrd,func.mkdir=pfrd,func.getattr=newest,minfreespace=20G,fsname=mergerfs 0 0
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
    having some of the files of a season, or audiobok or an album is the same
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
but only if the data on the other drives didn't change much as parity is
counted against content of all data drives.<br>
This behavior makes it good only for data that do not change often,
like movies, shows, music, photos, audiobooks, videos,...

Of note is that snapraid is saving parity information in to a single file.

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

  # autosave every 100GB
  autosave 100
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
  SNAPRAID_CONF="/etc/snapraid.conf"

  # -------------------------------------------------------------
  # Send ntfy push notification and log to journal function
  # -------------------------------------------------------------
  notify() {
      local msg="$1"
      printf "%b" "$msg" | /usr/bin/curl -s -d @- "$NTFY_TOPIC" || true
      echo -e "$msg" # To also log to systemd journal
  }

  echo "=== SnapRAID maintenance script started $(date +"%F %T") ==="
  
  # -------------------------------------------------------------
  # Check if if mergerfs mount point exists
  # -------------------------------------------------------------
  if ! mountpoint -q "$MERGERFS_MOUNT"; then
      notify "❌ mergerfs mount $MERGERFS_MOUNT not accessible!"
      exit 1
  fi

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
      while read -r disk; do
          disk=$(echo "$disk" | xargs) # Trim whitespace
          if [[ -e "$disk" && -b "$disk" ]]; then
              echo "$(date +"%F %T") SMART check for $disk"
              if ! smartctl -H "$disk" | grep -Eiq "PASSED|OK"; then
                  notify "❌ SMART health check failed for $disk"
              fi
          fi
      done < <(smartctl --scan | awk '{print $1}')
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
  `sudo systemctl enable --now snapraid-sync-and-maintenance.timer.timer`

</details>

---
---

<details>
<summary><h1>Network File Sharing - SMB and NFS</h1></summary>

### Samba

[Arch Wiki](https://wiki.archlinux.org/title/Samba)

* install samba
* copy the config below in to `/etc/samba/smb.conf`
* enable smb.service - `sudo systemctl enable --now smb.service`
* I dont install `nmb.service` for the old netbios discovery,
  it's dead technology.<br>
  If windows machines on the network **install wsdd** and enable
  the service `sudo systemctl enable --now wsdd.service`
* 

`/etc/samba/smb.conf`
```
[global]
   security = user
   map to guest = Bad User
   dns proxy = no
   disable netbios = yes
   smb ports = 445

[MergerFS]
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

</details>


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


# Other Guides

* https://perfectmediaserver.com/02-tech-stack/mergerfs/
* https://perfectmediaserver.com/03-installation/manual-install-ubuntu/
* https://thenomadcode.tech/mergerfs-snapraid-is-the-new-raid-5
* https://zackreed.me/snapraid-split-parity-sync-script/
* https://www.youtube.com/watch?v=Yt67zz9p0FU

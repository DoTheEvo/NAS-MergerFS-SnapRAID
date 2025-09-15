# NAS with MergerFS and SnapRAID

###### guide-by-example

WORK IN PROGRESS WORK IN PROGRESS WORK IN PROGRESS

![duf-pic](https://i.imgur.com/HRD6Al1.png)

# Overview

If one wants a NAS that allows mixed size HDDs, with parity, easy growth
and linux terminal is no problem.<br>
But beware - It is **so much more work** than just spinnig trueNAS or OMV.

### Pros:

  * Mixing HDDs of various sizes.
  * Trivial addition of more storage at any time.
  * Works at file level, where the files are spread across the disks,
    but perfectly accessible. Meaning you can litearlly pull a drive out,
    plug it into another PC and have full access to the data stored there.
  * Policies allow various methods how to spread files across the disks.
    Default being that files in the same directory end up on the same drive.
  * Power efficient, only the drive with the data being accessed spins up.
  * SnapRAID parity drive protects unlimited number of data drives.
  * Good performance.
  * Free and open source.

### Chapters:

* Linux and disks preparation
* MergerFS - merge disks in to one mount point
* SnapRAID - add a parity drive to prevent data loss on a disk failure
* Sharing the storage - SMB, NFS, iSCSI
* Spinning down the HDDs

## Linux and disks preparation

![linux-distros](https://i.imgur.com/fUUP3VA.png)

Have a linux installed on the machine.<br>
I use Arch, installing it using archinstall,
and have [an ansible playbooks](https://github.com/DoTheEvo/ansible-arch)
to set it up how I like it.

Now to format and partition the disks and mount them using fstab.

* Create a new partition **table** on each disk.<br>
  `sudo parted /dev/sdb --script mklabel gpt`<br>
  `sudo parted /dev/sdc --script mklabel gpt`<br>
  `sudo parted /dev/sdd --script mklabel gpt`<br>
* **Partition** the disks.<br>
  `sudo parted /dev/sdb --script mkpart primary ext4 0% 100%`<br>
  `sudo parted /dev/sdc --script mkpart primary ext4 0% 100%`<br>
  `sudo parted /dev/sdd --script mkpart primary ext4 0% 100%`<br>
* **Format** the partitions and label them.<br>
  `sudo mkfs.ext4 /dev/sdb1 -L disk_1`<br>
  `sudo mkfs.ext4 /dev/sdc1 -L disk_2`<br>
  `sudo mkfs.ext4 /dev/sdd1 -L disk_3`<br>
* Create **directories** where the drives will be mounted.<br>
  `sudo mkdir -p /mnt/disk_1 /mnt/disk_2 /mnt/disk_3`
* Edit **fstab** to mount the drives.<br>
  To make things easier, here’s a script that **generates the fstab entries**.<br>
  
  <details>
  <summary><h5>disks-fstab-entries.sh</h5></summary>

  ```bash
  #!/usr/bin/env bash
  # Generates a nicely formatted fstab for ext4 partitions and a MergerFS pool
  # Includes size and serial number for reference

  echo "# ======================================================"
  echo "# Individual ext4 disks"
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
  echo "# /mnt/disk_* /mnt/pool fuse.mergerfs defaults,allow_other,use_ino,category.create=epmfs 0 0"
  ```

  </details>

  It includes the size and the serial number of the disks,
  as well as a commented out mergerfs section that can be used later.
  * Make the script executable: `chmod +x disks-fstab-entries.sh`
  * Run it: `./disks-fstab-entries.sh`<br>
    It echoes stuff in to the terminal for you to copy/paste in to fstab.
  * Remove the lines you dont want, **edit the mount points** as needed
  since they all are set to `/mnt/disk_X` 

To mount all disks definied in fstab - `sudo mount -a` or just reboot.<br>
Check if all is fine with `lsblk` and `lsblk -f` and `duf` or `dysk`.

# MergerFS

![fstab-pic](https://i.imgur.com/bKhC1zT.png)

* [Github](https://github.com/trapexit/mergerfs)
* [The documentation.](https://trapexit.github.io/mergerfs/)

Merges disks of various sizes in to one pool.<br>

* Files are just simply spread across the disks and are **plainly accessible**
  even if a disk would be pulled out and placed in an another machine.
* Think of it as a **virtual mount point**, not a filesystem.
* Works at the **file level**, as oppose to the block level, and uses **fuse**
  in **user space**, as oppose to be living in the kernel space.
* There is **NO redundancy**, MergerFS is all about just merging disk space.
* Ideal for media storage - pictures, music, videos,... 
  for larger files that rarely change, as oppose to a millions of small files.
* Simple setup and configuration with a single line in `/etc/fstab`
* Easy to add drives, no rebuild time.
* Written in C++ and C.

### MergerFS setup

All your disks are formated, mounted and ready.

* Install mergerfs<br>
  `yay mergerfs`
* Create a directory for the merged mount point.<br>
  `sudo mkdir /mnt/pool`<br>
* edit the fstab, add mergerfs mount definition.<br>
  The `disk*` uses wildcard to catch all 
  ```
  /mnt/disk_* /mnt/pool fuse.mergerfs cache.files=partial,dropcacheonclose=true,ignorepponrename=true,allow_other,use_ino,category.create=epmfs,minfreespace=10G,fsname=mergerfs,defaults 0 0
  /mnt/disk_* /mnt/pool fuse.mergerfs defaults,allow_other,use_ino,category.create=epmfs 0 0
  ```
* Mount it.<br>
  `sudo mount /mnt/pool`
* Take ownership of the new mount `sudo chown $USER:$USER /mnt/pool`<br>

The fstab

# SnapRAID

* [Github](https://github.com/amadvance/snapraid)
* [Changelog](https://github.com/amadvance/snapraid/blob/master/HISTORY)
* [The documentation.](https://www.snapraid.it/manual)
* [Arch Wiki](https://wiki.archlinux.org/title/SnapRAID)

Provides protection against disk failure and bitrot.<br> 
Works at file level, not disk or block level.<br>
Unlike regular raid that is constant, snapraid needs to have scheduled runs,
usually once every 24 hours and you have ability to return to the state
from the last run.

Good for media storage - movies, shows, music, photos.

---

* install snapraid<br>
  `yay snapraid`
* Create `/etc/snapraid.conf`
  ```bash
  # Parity file
  parity /mnt/parity/snapraid.parity

  # Data disks
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

### SnapRAID Automation

* create a file `/opt/snapraid-sync-run-maint.sh`<br>
  ```bash
  #!/bin/bash
  set -euo pipefail

  LOG="/var/log/snapraid.log"
  exec >> "$LOG" 2>&1

  echo "=== SnapRAID job started at $(date) ==="

  # 1. Daily sync (skip files modified in last 30 min)
  /usr/bin/snapraid sync

  # 2. Monthly partial scrub (10%) on the 1st
  if [[ $(date +%d) -eq 01 ]]; then
      /usr/bin/snapraid scrub -p 10
  fi

  # 3. SMART check on the 1st of each month
  if [[ $(date +%d) -eq 15 ]]; then
      echo "--- SMART check ---"
      for disk in /dev/sd?; do
          echo "SMART for $disk"
          /usr/bin/smartctl -H "$disk"
      done
  fi

  echo "=== SnapRAID job finished at $(date) ==="
  ```

* Make it executable `sudo chmod +x /opt/snapraid-sync-run-maint.sh`
* create a systemd unit and a timer files.

  `/etc/systemd/system/snapraid-sync-run-maint.service`
  ```bash
  [Unit]
  Description=SnapRAID sync, scrub, smart

  [Service]
  Type=oneshot
  ExecStart=/opt/snapraid-sync-run-maint.sh
  ```

  `/etc/systemd/system/snapraid-sync-run-maint.timer`
  ```bash
  [Unit]
  Description=Run SnapRAID sync, scrub, smart,... nightly at 01:30

  [Timer]
  OnCalendar=*-*-* 01:30:00
  Persistent=true

  [Install]
  WantedBy=timers.target
  ```

* enable the timer<br>
  `sudo systemctl enable --now snapraid-sync-run-maint.timer.timer`

#### logrotate

To prevent growth of the log file.

* install `logrotate`
* create a config file<br>
  `/etc/logrotate.d/snapraid`
  ```bash
  /var/log/snapraid.log {
      size 20M
      rotate 1
      missingok
      notifempty
      create
  }
  ```
Can do dry run to see what it would do `sudo logrotate -d /etc/logrotate.conf`
</details>


### SnapRAID Notifications

I use [ntfy](https://github.com/DoTheEvo/selfhosted-apps-docker/tree/master/gotify-ntfy-signal)
for push notifications.

Create a new systemd unit file that will send the notification
to your ntfy server.

`/etc/systemd/system/ntfy@.service`
```bash
[Unit]
Description=ntfy notification service
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/curl --fail --retry 3 --retry-delay 30 -d "%i | %H" https://ntfy.example.com/snapraid
```

Edit the snapraid unit file, adding OnFailure and OnSuccess actions.

`snapraid-sync-run-maint.service`
```bash
[Unit]
Description=SnapRAID sync, scrub, smart
OnFailure=ntfy@failure-%p.service
OnSuccess=ntfy@success-%p.service

[Service]
Type=oneshot
ExecStart=/opt/snapraid-sync-run-maint.sh
```


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

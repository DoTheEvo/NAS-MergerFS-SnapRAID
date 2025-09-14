# NAS with mergerfs and snapraid

###### guide-by-example

WORK IN PROGRESS WORK IN PROGRESS WORK IN PROGRESS

# MergerFS

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
* Simple configuration with one line in `/etc/fstab`
* Easy to add drives, no rebuild time.
* Written in C++ and C.

## Preparations

Disks are formated and partitioned, mounted using fstab in to `/mnt`.

* Have a linux. My go-to is [Arch](https://github.com/DoTheEvo/ansible-arch).
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

To mount all disks definied in fstab - `sudo mount -a` or just reboot.
Check if all is fine with `lsblk` and `lsblk -f` and `duf`.


## MergerFS pooling

All your disks are formated, mounted and ready.

* Install mergerfs<br>
  `yay mergerfs`
* Create a directory for the merged drive.<br>
  `sudo mkdir /mnt/pool`<br>
* edit the fstab, uncoment or add a mergerfs mount definition.<br>
  ```
  /mnt/disk*  /mnt/pool  fuse.mergerfs cache.files=partial,dropcacheonclose=true,ignorepponrename=true,allow_other,use_ino,category.create=epmfs,minfreespace=10G,fsname=mergerfs,defaults 0 0
  ```
* Mount it.<br>
  `sudo mount /mnt/pool`
* Take ownership of the new mount `sudo chown $USER:$USER /mnt/pool`<br>



/mnt/disk*  /mnt/merger1  fuse.mergerfs cache.files=partial,dropcacheonclose=true,ignorepponrename=true,allow_other,use_ino,category.create=mfs,minfreespace=10G,fsname=mergerfs,defaults 0 0

# Guides

https://perfectmediaserver.com/02-tech-stack/mergerfs/
https://perfectmediaserver.com/03-installation/manual-install-ubuntu/



sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sdb1 -L disk_1
I like [cfdisk](https://i.imgur.com/6iwjRHE.mp4),

# SnapRAID

<details>
<summary><h1>Spinning down the HDDs</h1></summary>

![spindown-gif](https://i.imgur.com/QrhQWQc.gif)

Saves \~3W of power per disk, heat, wear and noise,... but the first time
accessing the storage takes \~10 seconds..

Unformated drives plugged will likely spin down on their own by their firmware,
but once mounted most distros do not spin down HDDs unless some configuration
is done.

What's required:

* Install `hdparm`
* Crete a new systemd service that on boot executes `hdparm -S 244`
  for every HDD.<br>
  Note that `244` [means](https://linux.die.net/man/8/hdparm) 2 hours.

`/etc/systemd/system/hdds-spindown-enabled.service`
```
[Unit]
Description=Enable spindown for all HDDs after 2 hours idle
After=local-fs.target systemd-udev-settle.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for d in /sys/block/sd*; do [ "$(cat $d/queue/rotational)" -eq 1 ] && /usr/bin/hdparm -S 244 /dev/$(basename $d); done'

[Install]
WantedBy=multi-user.target
```

Enable the service<br>
`sudo systemctl enable --now hdds-spindown-enabled.service`

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

* Create the logging script in `/opt/disks-spin-logger.sh`<br>
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
* Make the script executable: `chmod +x disks-spin-logger.sh`
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

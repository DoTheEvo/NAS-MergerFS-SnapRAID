# NAS with mergerfs and snapraid

###### guide-by-example

WORK IN PROGRESS WORK IN PROGRESS WORK IN PROGRESS

# MergerFS

* [Github](https://github.com/trapexit/mergerfs)
* [The documentation.](https://trapexit.github.io/mergerfs/)

Merges disks of various sizes in to one pool.<br>

* Files are just simply spread across disks and are **plainly accessible**
  even if a disk would be pulled out and placed in an another machine.
* Think of it as a **virtual mount point**, not a filesystem.
* Works at **file level**, as oppose to block level and uses **fuse**
  in **user space**, as oppose to be living in the kernel space.
* There is **NO redundancy**, MergerFS is all about just merging disk space.
* Has 
* Ideal for media storage - pictures, music, videos,... 
  for larger files that rarely change, as oppose to a millions of small files.
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
  To make it much easier, here's a script that spits out the lines for fstab.<br>
  Includes the size and the serial number of the disks, as well as commented
  out mergerfs section that will be used later.
  * Make the script executable: `chmod +x disks-fstab-entries.sh`
  * Run it: `./disks-fstab-entries.sh`
  * It echoes stuff in to the terminal for you to copy/paste in to fstab.
  * Remove the lines you dont want, edit the mount points as needed
  since they all are set to `/mnt/disk_X` 
  * To mount all disks definied in fstab - `sudo mount -a` or just reboot.

Check if all is fine with `lsblk` and `lsblk -f` and `duf`.

#### Spin down of disks

![spindown-gif](https://i.imgur.com/QrhQWQc.gif)

* *note* - Would run few weeks without spinning down.

Saves \~3W of power per disk, heat, wear and noise,... but the first time
accessing the storage takes \~10 seconds..<br> 

What's required:

* Each disk needs to be mounted with `x-systemd.automount` flag in fstab.<br>
  This unmounts disks on idle and mounts them only when access is requested.
* A hdparm -S <*time*> command needs to be executed on boot for each drive.<br>
  For this a systemd unit is created,
  note `242` [means](https://linux.die.net/man/8/hdparm) 1 hour.

`hdd-spindown.service`
```
[Unit]
Description=Sets spindown for all HDDs after 60min idle
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for d in /sys/block/sd*; do [ "$(cat $d/queue/rotational)" -eq 1 ] && /usr/bin/hdparm -S 242 /dev/$(basename $d); done'

[Install]
WantedBy=multi-user.target
```

Enable the service<br>
`sudo systemctl enable --now hdd-spindown.service`

A useful command showing the state of every drive:<br>
`for d in /dev/sd[a-z]; do echo -n "$d: "; sudo hdparm -C "$d" | grep state; done`


Be aware, freshly formated ext4 disks finish their initialization
in the background. Kernel runs `ext4lazyinit` process that zeroes inodes,
but one can hear disks chirping and doing something even when there should be
no activity. Can check if the process is running - `ps -ef | grep ext4lazyinit`

To see activity on a disk:

* `sudo iotop -ao`
* `sudo blktrace -d /dev/sdd -o - | blkparse -i -`

But also disks firmware might be doing stuff on its own.

* 41W idle but spinning
* 29W spin down

* 37W idle but spinning
* 29W idle but spinning

## MergerFS pooling

All your disks are formated, mounted and ready.

* Install mergerfs<br>
  `yay mergerfs`
* Create a directory for the merged drive.<br>
  `sudo mkdir /mnt/pool`<br>
* edit fstab uncoment, or add a mergerfs mount definition.<br>
  ```
  /mnt/disk*  /mnt/pool  fuse.mergerfs cache.files=partial,dropcacheonclose=true,ignorepponrename=true,allow_other,use_ino,category.create=epmfs,minfreespace=10G,fsname=mergerfs,defaults 0 0
  ```
* Mount it.<br>
  `sudo mount /mnt/pool`
* Take ownership of the new mount `sudo chown $USER:$USER /mnt/pool`<br>



# Trouble shooting

# Update

/mnt/disk*  /mnt/merger1  fuse.mergerfs cache.files=partial,dropcacheonclose=true,ignorepponrename=true,allow_other,use_ino,category.create=mfs,minfreespace=10G,fsname=mergerfs,defaults 0 0

# Guides

https://perfectmediaserver.com/03-installation/manual-install-ubuntu/



sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sdb1 -L disk_1
I like [cfdisk](https://i.imgur.com/6iwjRHE.mp4),

# SnapRAID

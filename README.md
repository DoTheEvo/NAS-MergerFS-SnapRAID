# NAS with mergerfs and snapraid

###### guide-by-example

WORK IN PROGRESS WORK IN PROGRESS WORK IN PROGRESS

# MergerFS

* [Github](https://github.com/trapexit/mergerfs)
* [The documentation.](https://trapexit.github.io/mergerfs/)

Merges disks of various sizes in to one filesystem, one mount point.<br>

* Think of it as a **virtual mount point**, not a filesystem.
* Works at **file level**, as oppose to being on block level.
* Uses **fuse** in **userspace**, as oppose to be living in kernel space.
* Files are just simply spread across disks and are plainly accessible
  even if a disk would be pulled out and placed in another machine.
* Ideal for media storage - pictures, music, videos,... rarely changing files,
  as oppose to a millions of small files.

### The Setup

This setup 

* Install linux.<br>
  My go-to is [arch](https://github.com/DoTheEvo/ansible-arch).
* partition the disks you plan to use,
  `sudo parted /dev/sdb --script mklabel gpt`
  `sudo parted /dev/sdc --script mklabel gpt`
  `sudo parted /dev/sdd --script mklabel gpt`

* format the disks with your prefered filesystem, ext4 is used here.<br>
  `sudo mkfs.ext4 /dev/sdb1`<br>
  `sudo mkfs.ext4 /dev/sdc1`<br>
  `sudo mkfs.ext4 /dev/sdd1`<br>
* Prepare directories to mount the disks.<br>
  `sudo mkdir /mnt/parity1`<br>
  `sudo mkdir /mnt/disk1`<br>
  `sudo mkdir /mnt/disk2`<br>
* Mount the disks with noatime and nofail option.<br>
  `sudo mount -o noatime,nofail /dev/sdb1 /mnt/parity1`<br>
  `sudo mount -o noatime,nofail /dev/sdc1 /mnt/disk1`<br>
  `sudo mount -o noatime,nofail /dev/sdd1 /mnt/disk2`<br>
* Backup the original fstab. Generate new fstab using genfstab script,
  check if it looks ok, compare with the original, remove zram or any stuff
  that is not suppose to be there.
  Replace the original fstab with the fstab, we make backup first<br>
  `sudo cp /etc/fstab /etc/fstab.bak`<br>
  `sudo genfstab -U / > ~/fstab`<br>
  `sudo cp ~/fstab /etc/fstab`<br>


Reboot, check if all is fine with `lsblk` and `lsblk -f` and `duf`.<br>
What we should have is clean linux with 3 ext4 drives mounted in /mnt

# mergerfs

* Install mergerfs<br>
  `yay mergerfs`
* Create a directory for the merged drive.<br>
  `sudo mkdir /mnt/merger1`<br>
  `sudo chown $USER:$USER /mnt/merger1`<br>
* edit fstab adding a mergerfs mount.<br>
  ```
  /mnt/disk*  /mnt/merger1  fuse.mergerfs cache.files=partial,dropcacheonclose=true,ignorepponrename=true,allow_other,use_ino,category.create=mfs,minfreespace=10G,fsname=mergerfs,defaults 0 0
  ```
* Mount it.<br>
  `sudo mount /mnt/merger1`



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

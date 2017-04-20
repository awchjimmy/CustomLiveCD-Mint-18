# LiveCDCustomization
```
原文教學：
https://help.ubuntu.com/community/LiveCDCustomization
```

```
<!> 本教學只適用 Ubuntu 類發行版
<!> Host 跟 LiveCD 架構必須相同 (x86_64)
```

---

## Install pre-requisities
```
$ sudo apt install squashfs-tools genisoimage
```

## Obtain the base system
```
$ mkdir ~/livecdtmp
$ cp linuxmint-18.1-xfce-64bit.iso ~/livecdtmp
$ cd ~/livecdtmp
```

### Extract the CD .iso contents
Mount the Desktop .iso
```
$ mkdir mnt
$ sudo mount -o loop linuxmint-18.1-xfce-64bit.iso mnt
```
Extract .iso contents into dir 'extract-cd'
```
$ mkdir extract-cd
$ sudo rsync --exclude=/casper/filesystem.squashfs -a mnt/ extract-cd
```

### Extract the Desktop system
```
$ sudo unsquashfs mnt/casper/filesystem.squashfs
$ sudo mv squashfs-root edit
```

### Prepare and chroot
```
<!> If you do this in 14.04 LTS, you will lose network connectivity (name resolving part of it)
```
On more recent releases, you can avoid this issue by just binding /run instead, which will pull your host's resolvconf info into the chroot:
```
$ sudo mount -o bind /run/ edit/run
```
```
$ sudo mount --bind /dev/ edit/dev
$ sudo chroot edit
$ mount -t proc none /proc
$ mount -t sysfs none /sys
$ mount -t devpts none /dev/pts
```
To avoid locale issues and in order to import GPG keys
```
$ export HOME=/root
$ export LC_ALL=C
```

## Customizations
```
$ apt-get update
$ apt-get install package-name
```

To view installed packages by size
```
$ dpkg-query -W --showformat='${Installed-Size}\t${Package}\n' | sort -nr | less
```

When you want to remove packages remember to use purge
```
$ apt-get purge package-name
```

## Cleanup
Be sure to remove any temporary files which are no longer needed, as space on a CD is limited. A classic example is downloaded package files, which can be cleaned out using:
```
$ apt-get clean
```
Or delete temporary files
```
$ rm -rf /tmp/* ~/.bash_history
```
from within the chroot environment.
now umount (unmount) special filesystems and exit chroot
```
$ umount /proc || umount -lf /proc
$ umount /sys
$ umount /dev/pts
$ exit
$ sudo umount edit/dev
```

## Producing the CD image
### Assembling the file system
Regenerate manifest
```
$ sudo chmod +w extract-cd/casper/filesystem.manifest
$ sudo su
$ chroot edit dpkg-query -W --showformat='${Package} ${Version}\n' > extract-cd/casper/filesystem.manifest
$ exit
$ sudo cp extract-cd/casper/filesystem.manifest extract-cd/casper/filesystem.manifest-desktop
$ sudo sed -i '/ubiquity/d' extract-cd/casper/filesystem.manifest-desktop
$ sudo sed -i '/casper/d' extract-cd/casper/filesystem.manifest-desktop
```
Compress filesystem
```
$ sudo mksquashfs edit extract-cd/casper/filesystem.squashfs
```
Update the filesystem.size file, which is needed by the installer:
```
$ sudo su
$ printf $(du -sx --block-size=1 edit | cut -f1) > extract-cd/casper/filesystem.size
$ exit
```
Remove old md5sum.txt and calculate new md5 sums
```
$ cd extract-cd
$ sudo rm MD5SUMS
$ find -type f -print0 | sudo xargs -0 md5sum | grep -v isolinux/boot.cat | sudo tee MD5SUMS
```
Create the ISO image
```
$ sudo mkisofs -D -r -V "mint 18.1 x64" -cache-inodes -J -l -b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -o ../linuxmint-18.1-xfce-x86_64-custom.iso .
```

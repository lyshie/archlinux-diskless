# archlinux-diskless
ArchLinux 無硬碟系統 (遠端派送)

## 客戶端模式
- NFS 模式提供讀寫，可從遠端進行系統更新
- NBD 模式提供唯讀，無硬碟系統開機自動還原

## 安裝客戶端作業系統 (Client)
- 建立硬碟空間 sparse file
```
# truncate -s 10G /srv/arch.img
```
- 建立 btrfs 檔案系統
```
# mkfs.btrfs /srv/arch.img
```
- 掛載硬碟
```
# export root=/srv/arch
# mkdir -p "$root"
# mount -o loop,discard,compress=lzo /srv/arch.img "$root"
```
- 由 Host 端安裝 Guest 的作業系統
```
# pacstrap -d "$root" base mkinitcpio-nfs-utils nfs-utils
```
- NFS (如有使用 NFSv4)
```
# sed s/nfsmount/mount.nfs4/ "$root/usr/lib/initcpio/hooks/net" > "$root/usr/lib/initcpio/hooks/net_nfs4"
# cp $root/usr/lib/initcpio/install/net{,_nfs4}
```
- 增加所需的驅動程式 (網路界面)
```
# vim $root/etc/mkinitcpio.conf
MODULES="nfs nfsv4 nbd r8168 broadcom tg3 e1000 e1000e"
BINARIES="/usr/bin/mount.nfs /usr/bin/mount.nfs4"
HOOKS="base udev autodetect modconf net nbd block filesystems keyboard fsck"
```
- NBD (手動安裝 mkinitcpio-nbd)
```
# pacman --root "$root" --dbpath "$root/var/lib/pacman" -U mkinitcpio-nbd-0.4-1-any.pkg.tar.xz
```
- 設定 GRUB 開機程式
```
# pacman --root "$root" --dbpath "$root/var/lib/pacman" -S grub
# arch-chroot "$root" grub-mknetdir --net-directory=/boot --subdir=grub
# vim "$root/boot/grub/grub.cfg"
menuentry "Arch Linux (NBD)" {
    linux /vmlinuz-linux quiet add_efi_memmap ip=:::::eth0:dhcp nbd_host=163.26.68.15 nbd_name=arch root=/dev/nbd0
    initrd /initramfs-linux.img
}
menuentry "Arch Linux (NFS)" {
    linux /vmlinuz-linux quiet add_efi_memmap ip=:::::eth0:dhcp nfsroot=163.26.68.15:/srv/arch,v3 rootfstype=nfs
    initrd /initramfs-linux.img
}
```
- NBS root
```
# vim "$root/etc/fstab"
/dev/nbd0  /  btrfs  rw,noatime,nodiratime,discard,compress=lzo  0 0
```

## 參考文件
- [ArchLinux - Diskless system](https://wiki.archlinux.org/index.php/Diskless_system)

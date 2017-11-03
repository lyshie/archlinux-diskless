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
- 贈加所需的驅動程式 (網路界面)
```
# vim $root/etc/mkinitcpio.conf
MODULES="nfs nfsv4 nbd r8168 broadcom tg3 e1000 e1000e"
BINARIES="/usr/bin/mount.nfs /usr/bin/mount.nfs4"
HOOKS="base udev autodetect modconf net nbd block filesystems keyboard fsck"
```

## 參考文件
- [ArchLinux - Diskless system](https://wiki.archlinux.org/index.php/Diskless_system)

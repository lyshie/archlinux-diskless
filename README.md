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
tmpfs    /home/student/.cache/chromium  tmpfs noatime,nodiratime,nodev,nosuid,noexec,uid=student,gid=student,noauto,comment=systemd.automount
tmpfs   /var/log        tmpfs     nodev,nosuid    0 0
tmpfs   /var/spool/cups tmpfs     nodev,nosuid    0 0
```
- 更新後打包映像檔
```bash
#!/bin/sh

arch-chroot /srv/arch mkinitcpio -p linux

btrfs filesystem defragment -r -v /srv/arch
find /srv/arch -xdev -type d -print -exec btrfs filesystem defragment '{}' \;
sync

systemctl stop nfs-server
systemctl stop nbd-server
systemctl stop dhcpd4
systemctl stop tftpd
sync

umount /srv/arch    # arch.img 線上使用

cp -p /srv/arch2.img /srv/arch3.img    # arch3.img 備份既有的映像檔
sync

cp /srv/arch.img /srv/arch2.img    # arch2.img 提供給 NBD 使用
sync
```

## 伺服器設定
- 設定 DHCP，自動分配 IP 位址與伺服器資訊
```
# vim /etc/dhcpd.conf
allow booting;
allow bootp;

authoritative;

option domain-name-servers 163.26.68.1;
option routers 163.26.69.254;
option subnet-mask 255.255.254.0;

option architecture code 93 = unsigned integer 16;

subnet 163.26.68.0 netmask 255.255.254.0 {
    next-server 163.26.68.15;

    if option architecture = 00:07 {
        filename "/grub/x86_64-efi/core.efi";
        #filename "/grub/i386-pc/core.0";
    } else {
        filename "/grub/i386-pc/core.0";
    }

    pool {
        next-server 163.26.68.15;
        #filename "/syslinux/pxelinux.0";
        range 163.26.69.45 163.26.69.90;
    }

    group {
        next-server 163.26.68.15;
        #filename "/syslinux/pxelinux.0";
        include "/etc/pcroom.conf";
    }
}
```
```
# vim /etc/pcroom.conf
host PC01 {
    hardware ethernet C0:3F:D5:B7:1C:8E;
    fixed-address 163.26.69.1;
}

host PC02 {
    hardware ethernet C0:3F:D5:B7:27:8A;
    fixed-address 163.26.69.2;
}
```
- 設定 TFTP 伺服器，派送開機程式 (grub, syslinux)、核心 (vmlinuz) 與檔案系統 (initramfs)
```
# pacman -S tftp-hpa
# vim /etc/conf.d/tftpd
TFTPD_ARGS="--secure /srv/arch/boot"    # 映像檔掛載後直接使用
```
- 設定 NBD 伺服器，提供無硬碟系統
```
# vim /etc/nbd-server/config
[generic]
    listenaddr = 0.0.0.0
[arch]
    exportname = /srv/arch2.img
    copyonwrite = true
```
- 設定 NFS 伺服器，提供與端更新作業系統
```
# vim /etc/exports
/srv/arch  *(rw,no_root_squash,no_subtree_check)
```

## GUI 環境設定
- LightDM 自動登入
```
# vim /srv/arch/etc/lightdm/lightdm.conf 
[Seat:*]
autologin-user=student
```
## 進階設定 (arch.img)
```
# useradd student
# mkdir /home/student
# passwd student

# vim /etc/netctl/eth0
Description='A basic dhcp ethernet connection'
Interface=eth0
Connection=ethernet
IP=dhcp
DNS=('163.26.68.1')
# systemctl enable netctl
# netctl enable eth0

# pacman -Sy lightdm
# groupadd -r autologin
# gpasswd -a student autologin
# chown -R admin:admin /home/admin
# chown -R student:student /home/student/

# vim /home/student/.xprofile
export VTE_BACKEND=Pango
export OOO_FORCE_DESKTOP=gnome
export GDK_USE_XFT=1
export QT_XFT=true
export MOZ_DISABLE_PANGO=1

export LANG="zh_TW.UTF-8"
export LC_CTYPE="zh_TW.UTF-8"
export XMODIFIERS=@im=gcin
export GTK_IM_MODULE="gcin"
export QT_IM_MODULE="gcin"
gcin &

# pacman -Sy sudo
# usermod -a -G wheel student
# vim /etc/sudoers
%wheel ALL=(ALL) ALL

# vim /etc/autofs/auto.misc
106     -fstype=cifs,username=username,password=password  ://163.26.68.2/106學生作品
```

## 參考文件
- [ArchLinux - Diskless system](https://wiki.archlinux.org/index.php/Diskless_system)

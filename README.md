# archlinux-diskless
ArchLinux 無硬碟系統 (遠端派送)

## 安裝客戶端作業系統 (Client)
- 建立硬碟空間 sparse file
```
# truncate -s 10G /srv/arch.img
```
- 建立 btrfs 檔案系統
```
# mkfs.btrfs /srv/arch.img
```

## 參考文件
- [ArchLinux - Diskless system](https://wiki.archlinux.org/index.php/Diskless_system)

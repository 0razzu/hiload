# Linux Архитектура и файловые системы

## Задание 1. Kernel and Module Inspection

Ядро:
```
dmitry@host1:~$ uname -r
6.8.0-85-generic
```

Его модули:
```
dmitry@host1:~$ lsmod
Module                  Size  Used by
tls                   159744  0
qrtr                   49152  2
cfg80211             1249280  0
binfmt_misc            28672  1
snd_hda_codec_idt      69632  1
snd_hda_codec_generic   114688  1 snd_hda_codec_idt
snd_hda_intel          57344  0
snd_intel_dspcfg       20480  1 snd_hda_intel
nls_iso8859_1          12288  1
snd_hda_codec         208896  3 snd_hda_codec_generic,snd_hda_intel,snd_hda_codec_idt
snd_hda_core          163840  4 snd_hda_codec_generic,snd_hda_intel,snd_hda_codec,snd_hda_codec_idt
snd_hwdep              24576  1 snd_hda_codec
snd_pcm               196608  3 snd_hda_intel,snd_hda_codec,snd_hda_core
vmwgfx                446464  0
snd_timer              53248  1 snd_pcm
snd                   147456  7 snd_hda_codec_generic,snd_hwdep,snd_hda_intel,snd_hda_codec,snd_timer,snd_pcm,snd_hda_codec_idt
drm_ttm_helper         12288  1 vmwgfx
soundcore              16384  1 snd
ttm                   106496  2 vmwgfx,drm_ttm_helper
input_leds             12288  0
joydev                 36864  0
uio_pdrv_genirq        16384  0
uio                    32768  1 uio_pdrv_genirq
sch_fq_codel           24576  3
dm_multipath           49152  0
efi_pstore             12288  0
nfnetlink              20480  2
ip_tables              36864  0
x_tables               65536  1 ip_tables
autofs4                57344  2
btrfs                1929216  0
blake2b_generic        24576  0
raid10                 77824  0
raid456               212992  0
async_raid6_recov      24576  1 raid456
async_memcpy           16384  2 raid456,async_raid6_recov
async_pq               16384  2 raid456,async_raid6_recov
async_xor              16384  3 async_pq,raid456,async_raid6_recov
async_tx               16384  5 async_pq,async_memcpy,async_xor,raid456,async_raid6_recov
xor                    12288  2 async_xor,btrfs
xor_neon               16384  1 xor
raid6_pq              110592  4 async_pq,btrfs,raid456,async_raid6_recov
libcrc32c              12288  2 btrfs,raid456
raid1                  61440  0
raid0                  24576  0
hid_generic            12288  0
usbhid                 81920  0
hid                   184320  2 usbhid,hid_generic
crct10dif_ce           12288  1
polyval_ce             12288  0
polyval_generic        12288  1 polyval_ce
ghash_ce               24576  0
sm4                    12288  0
sha3_ce                16384  0
sha2_ce                20480  0
sha256_arm64           24576  1 sha2_ce
sha1_ce                12288  0
e1000                 221184  0
gpio_keys              24576  0
xhci_pci               28672  0
xhci_pci_renesas       24576  1 xhci_pci
aes_neon_bs            24576  0
aes_neon_blk           28672  1 aes_neon_bs
aes_ce_blk             36864  0
aes_ce_cipher          12288  1 aes_ce_blk
```

Отключаю автозагрузку cdrom (на всякий случай сначала проверил, что нет файла с таким именем и что формат конфига такой, какой я ожидаю):
```
dmitry@host1:~$ ls /etc/modprobe.d
blacklist-ath_pci.conf  blacklist.conf  blacklist-firewire.conf  blacklist-framebuffer.conf  blacklist-rare-network.conf  iwlwifi.conf  mdadm.conf
dmitry@host1:~$ cat /etc/modprobe.d/blacklist.conf
# This file lists those modules which we don't want to be loaded by
# alias expansion, usually so some other driver will be loaded for the
# device instead.

# evbug is a debug tool that should be loaded explicitly
blacklist evbug

# these drivers are very simple, the HID drivers are usually preferred
blacklist usbmouse
blacklist usbkbd

# replaced by e100
blacklist eepro100

# replaced by tulip
blacklist de4x5

# causes no end of confusion by creating unexpected network interfaces
blacklist eth1394

# snd_intel8x0m can interfere with snd_intel8x0, doesn't seem to support much
# hardware on its own (Ubuntu bug #2011, #6810)
blacklist snd_intel8x0m

# Conflicts with dvb driver (which is better for handling this device)
blacklist snd_aw2

# replaced by p54pci
blacklist prism54

# replaced by b43 and ssb.
blacklist bcm43xx

# most apps now use garmin usb driver directly (Ubuntu: #114565)
blacklist garmin_gps

# replaced by asus-laptop (Ubuntu: #184721)
blacklist asus_acpi

# low-quality, just noise when being used for sound playback, causes
# hangs at desktop session start (Ubuntu: #246969)
blacklist snd_pcsp

# ugly and loud noise, getting on everyone's nerves; this should be done by a
# nice pulseaudio bing (Ubuntu: #77010)
blacklist pcspkr

# EDAC driver for amd76x clashes with the agp driver preventing the aperture
# from being initialised (Ubuntu: #297750). Blacklist so that the driver
# continues to build and is installable for the few cases where its
# really needed.
blacklist amd76x_edac
dmitry@host1:~$ sudo bash -c "echo 'blacklist cdrom' > /etc/modprobe.d/blacklist-cdrom.conf"
[sudo] password for dmitry: 
dmitry@host1:~$ ls /etc/modprobe.d
blacklist-ath_pci.conf  blacklist-cdrom.conf  blacklist.conf  blacklist-firewire.conf  blacklist-framebuffer.conf  blacklist-rare-network.conf  iwlwifi.conf  mdadm.conf
```

Конфигурация ядра. Интересует нас файлик `/boot/config-$(uname -r)`. В нём key–value пары «фича—режим», где режим — `y` для встроенных в ядро, `m` для скомпилированных как модуль, `n` для отключённых (их также просто прячут в коммент). Сам файлик — та ещё простыня, поэтому грепаю.
```
dmitry@host1:~$ cat /boot/config-$(uname -r) | grep CONFIG_XFS_FS
CONFIG_XFS_FS=m
```

У меня поддержка XFS включена модулем.


## Задание 2. Наблюдение за VFS

Запускаю `strace` с инспектированием системных вызовов для следующих операций с файлами: открытие, чтение, запись и закрытие:
```
dmitry@host1:~$ strace -e trace=openat,read,write,close cat /etc/os-release > /dev/null
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
close(3)                                = 0
openat(AT_FDCWD, "/lib/aarch64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0\267\0\1\0\0\0\360\206\2\0\0\0\0\0"..., 832) = 832
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
close(3)                                = 0
openat(AT_FDCWD, "/etc/os-release", O_RDONLY) = 3
read(3, "PRETTY_NAME=\"Ubuntu 24.04.3 LTS\""..., 131072) = 400
write(1, "PRETTY_NAME=\"Ubuntu 24.04.3 LTS\""..., 400) = 400
read(3, "", 131072)                     = 0
close(3)                                = 0
close(1)                                = 0
close(2)                                = 0
+++ exited with 0 +++
```

Читаем вывод по порядку:
1. `openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3` — открываем каталог кэша динамических библиотек в режиме readonly с закрытием при выполнении `exec()`. Нужно для работы `cat`: как дальше увидим, ему нужна `libc`. А `AT_FDCWD` — дескриптор рабочей директории, т. е. был бы путь относительным — открыли бы относительно cwd.
2. `close(3)` — сразу закрываем каталог кэша; дальше ещё парочка таких вызовов — скипну их.
3. `openat(AT_FDCWD, "/lib/aarch64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3` — теперь открываем библиотечку `libc`.
5. `read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0\267\0\1\0\0\0\360\206\2\0\0\0\0\0"..., 832) = 832` — собственно чтение `libc`, ELF как раз про формат бинарника.
6. `openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3` — открываем архив локализации.
7. `openat(AT_FDCWD, "/etc/os-release", O_RDONLY) = 3` — наконец открываем файл из команды.
8. `read(3, "PRETTY_NAME=\"Ubuntu 24.04.3 LTS\""..., 131072) = 400` — читаем из него 400 б.
9. `write(1, "PRETTY_NAME=\"Ubuntu 24.04.3 LTS\""..., 400) = 400` — пишем их в stdout.
10. `read(3, "", 131072)                     = 0` — пытаемся прочитать ещё, но читать уже нечего.
11. Закрываем файл, stdout и stderr.

Читали мы файлик с информацией о релизе ОС. В него записаны в key–value-формате такие характеристики как название дистрибутива, версия, сходство с другими дистрибутивами и ссылки на страницы, связанные с дистрибутивом. Нужен, чтобы программы знали, на чём запускаются.

Записывающий вызов всё-таки присутствует в выводе. Вылез он потому, что `cat` реально пишет в stdout, и лишь потом shell перенаправляет в `/dev/null` — `cat` про перенаправления не думает и пишет, куда пишет по дефолту; специальной предобработки, чтобы соптимизировать наш особый случай, нет (да и как будто не unix way).


## Задание 3. LVM Management

Глянем сначала, что сейчас по дискам у нас:
```
dmitry@host1:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   20G  0 disk 
├─sda1                      8:1    0  953M  0 part /boot/efi
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0 17.3G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0   10G  0 lvm  /
sr0                        11:0    1 1024M  0 rom
```

Добавляю, смотрю ещё раз:
```
dmitry@host1:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   20G  0 disk 
├─sda1                      8:1    0  953M  0 part /boot/efi
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0 17.3G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0   10G  0 lvm  /
sdb                         8:16   0    2G  0 disk 
sr0                        11:0    1 1024M  0 rom
```

Создаю раздел и проверяю:
```
dmitry@host1:~$ sudo fdisk /dev/sdb
[sudo] password for dmitry: 

Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x67f53cdf.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-4194303, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-4194303, default 4194303): 

Created a new partition 1 of type 'Linux' and of size 2 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

dmitry@host1:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   20G  0 disk 
├─sda1                      8:1    0  953M  0 part /boot/efi
├─sda2                      8:2    0  1.8G  0 part /boot
└─sda3                      8:3    0 17.3G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0   10G  0 lvm  /
sdb                         8:16   0    2G  0 disk 
└─sdb1                      8:17   0    2G  0 part 
sr0                        11:0    1 1024M  0 rom
```

Теперь Physical Volume:
```
dmitry@host1:~$ sudo pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
dmitry@host1:~$ sudo pvs
  PV         VG        Fmt  Attr PSize   PFree 
  /dev/sda3  ubuntu-vg lvm2 a--  <17.32g <7.32g
  /dev/sdb1            lvm2 ---   <2.00g <2.00g
```

И Volume Group:
```
dmitry@host1:~$ sudo vgcreate vg_highload /dev/sdb1
  Volume group "vg_highload" successfully created
dmitry@host1:~$ sudo vgs
  VG          #PV #LV #SN Attr   VSize   VFree 
  ubuntu-vg     1   1   0 wz--n- <17.32g <7.32g
  vg_highload   1   0   0 wz--n-  <2.00g <2.00g
```

Logical Volumes:
```
dmitry@host1:~$ sudo lvcreate -L 1200M -n data_lv vg_highload
  Logical volume "data_lv" created.
dmitry@host1:~$ sudo lvcreate -l 100%FREE -n logs_lv vg_highload
  Logical volume "logs_lv" created.
dmitry@host1:~$ sudo lvs
  LV        VG          Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  ubuntu-lv ubuntu-vg   -wi-ao----  10.00g                                                    
  data_lv   vg_highload -wi-a-----   1.17g                                                    
  logs_lv   vg_highload -wi-a----- 844.00m
```

Форматирую и монтирую:
```
dmitry@host1:~$ sudo mkfs.ext4 /dev/vg_highload/data_lv
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 307200 4k blocks and 76800 inodes
Filesystem UUID: 86da08c5-9eac-4a56-86ae-ebdc88364b48
Superblock backups stored on blocks: 
    32768, 98304, 163840, 229376, 294912

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done 

dmitry@host1:~$ sudo mkdir -p /mnt/app_data
dmitry@host1:~$ sudo mount /dev/vg_highload/data_lv /mnt/app_data
dmitry@host1:~$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              391M  1.0M  390M   1% /run
efivarfs                           256K   89K  168K  35% /sys/firmware/efi/efivars
/dev/mapper/ubuntu--vg-ubuntu--lv  9.8G  4.1G  5.2G  45% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          1.7G  102M  1.5G   7% /boot
/dev/sda1                          952M  6.4M  945M   1% /boot/efi
tmpfs                              390M   12K  390M   1% /run/user/1000
/dev/mapper/vg_highload-data_lv    1.2G   24K  1.1G   1% /mnt/app_data
dmitry@host1:~$ sudo mkfs.xfs /dev/vg_highload/logs_lv
meta-data=/dev/vg_highload/logs_lv isize=512    agcount=4, agsize=54016 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=216064, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
dmitry@host1:~$ sudo mkdir -p /mnt/app_logs
dmitry@host1:~$ sudo mount /dev/vg_highload/logs_lv /mnt/app_logs
dmitry@host1:~$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              391M  1.0M  390M   1% /run
efivarfs                           256K   89K  168K  35% /sys/firmware/efi/efivars
/dev/mapper/ubuntu--vg-ubuntu--lv  9.8G  4.1G  5.2G  45% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          1.7G  102M  1.5G   7% /boot
/dev/sda1                          952M  6.4M  945M   1% /boot/efi
tmpfs                              390M   12K  390M   1% /run/user/1000
/dev/mapper/vg_highload-data_lv    1.2G   24K  1.1G   1% /mnt/app_data
/dev/mapper/vg_highload-logs_lv    780M   48M  733M   7% /mnt/app_logs
```


## Задание 4. Использование pseudo filesystem

Читаю каталог `/proc`, нахожу нужные файлики, читаю:
```
dmitry@host1:~$ cat /proc/cpuinfo
processor    : 0
BogoMIPS    : 48.00
Features    : fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm sb paca pacg dcpodp flagm2 frint bf16 afp
CPU implementer    : 0x61
CPU architecture: 8
CPU variant    : 0x0
CPU part    : 0x000
CPU revision    : 0

processor    : 1
BogoMIPS    : 48.00
Features    : fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm sb paca pacg dcpodp flagm2 frint bf16 afp
CPU implementer    : 0x61
CPU architecture: 8
CPU variant    : 0x0
CPU part    : 0x000
CPU revision    : 0

processor    : 2
BogoMIPS    : 48.00
Features    : fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm sb paca pacg dcpodp flagm2 frint bf16 afp
CPU implementer    : 0x61
CPU architecture: 8
CPU variant    : 0x0
CPU part    : 0x000
CPU revision    : 0

processor    : 3
BogoMIPS    : 48.00
Features    : fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm sb paca pacg dcpodp flagm2 frint bf16 afp
CPU implementer    : 0x61
CPU architecture: 8
CPU variant    : 0x0
CPU part    : 0x000
CPU revision    : 0

```

Похоже, что у меня модель CPU — большой секрет. С памятью приятнее:
```
dmitry@host1:~$ cat /proc/meminfo
MemTotal:        3993632 kB
...
```

Чуть более настоичиво с процессором (всё равно точную модель не отдаёт):
```
dmitry@host1:~$ lscpu
Architecture:             aarch64
  CPU op-mode(s):         64-bit
  Byte Order:             Little Endian
CPU(s):                   4
  On-line CPU(s) list:    0-3
Vendor ID:                Apple
  Model name:             -
    Model:                0
    Thread(s) per core:   1
    Core(s) per cluster:  4
    Socket(s):            -
    Cluster(s):           1
    Stepping:             0x0
    BogoMIPS:             48.00
    Flags:                fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma lrcpc dcpop sha3 asimddp sha512 asimdfhm dit uscat ilrcpc flagm sb paca pacg dcpodp flagm2 frint 
                          bf16 afp
NUMA:                     
  NUMA node(s):           1
  NUMA node0 CPU(s):      0-3
Vulnerabilities:          
  Gather data sampling:   Not affected
  Itlb multihit:          Not affected
  L1tf:                   Not affected
  Mds:                    Not affected
  Meltdown:               Not affected
  Mmio stale data:        Not affected
  Reg file data sampling: Not affected
  Retbleed:               Not affected
  Spec rstack overflow:   Not affected
  Spec store bypass:      Vulnerable
  Spectre v1:             Mitigation; __user pointer sanitization
  Spectre v2:             Not affected
  Srbds:                  Not affected
  Tsx async abort:        Not affected
```

Ppid текущего shell:
```
dmitry@host1:~$ cat /proc/$$/status | grep PPid
PPid:    1064
```

`$$` — переменная shell, содержащая его pid.

Настройки I/O Scheduler для `/dev/sda`:
```
dmitry@host1:~$ cat /sys/block/sda/queue/scheduler
[none] mq-deadline
```

Текущий планировщик оборачивается в скобки, т. е. у меня отключён. Доступен `mq-deadline`.

Определяю размер MTU для `enp0s8`:
```
dmitry@host1:~$ cat /sys/class/net/enp0s8/mtu
1500
```

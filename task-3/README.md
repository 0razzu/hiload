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

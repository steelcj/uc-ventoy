# uc-ventoy troubleshooting 

## To get the stick written

### Unmount and unplug the stick

Unmount and unplug the stick, then plug it in again, in another port if you can.

### Check the file system

For example on an exfat (default) file system funr it against  /dev/sdX1 (the device name may change).

```bash
sudo fsck.exfat /dev/sdb1
exfatprogs version : 1.2.2
/dev/sdb1: clean. directories 8, files 11
```

### Compare checksums

Compare the ISOs on the stick with your downloaded copies, for example with sha256sum, and recopy any that differ.

### Run bin/build

```bash
bin/build --stick /media/initial/Ventoy
```

### Unmount and replug

Unmount, replug, and look at the files; check journalctl -k for errors on the device.

```bash
journalctl -k | grep sdb
journalctl -k | grep sdc1
```


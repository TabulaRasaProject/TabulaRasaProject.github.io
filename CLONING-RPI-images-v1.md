# Options for Cloning or Backing Up the Raspberry Pi

## 1. Full Disk Image (Bootable Snapshot)
This creates a complete copy of your Pi’s Micro/SDCard/SSD that you can restore later. From another Linux machine (or the Pi itself).

Identify your SD card device (e.g. /dev/sda, /dev/mmcblk0):
```
lsblk
```

Backup to an image file
```
sudo dd if=/dev/mmcblk0 of=~/rpi_backup.img bs=4M status=progress
```
Compress it (saves space)
```
xz -z ~/rpi_backup.img
```
To restore:
```
xz -d ~/rpi_backup.img.xz
sudo dd if=~/rpi_backup.img of=/dev/mmcblk0 bs=4M status=progress
```
⚠️ Careful: Mixing up if/of will wipe your drive.

## 2. Incremental / Snapshot-style Backups
  If you don’t need the entire OS image, just your work and configs use Rsync (efficient, incremental):
```
rsync -avh --delete /home/pi/ user@backupserver:/backups/pi-home/
rsync -avh --delete /etc/ user@backupserver:/backups/pi-etc/
```
BONUS: add this to a cron job for daily/weekly backups.


## 3. Specialized Tools
Clones the running system live to /dev/sda:
```
rpi-clone → makes a bootable clone to another USB stick or drive.
git clone https://github.com/billw2/rpi-clone.git
sudo ./rpi-clone -f /dev/sda
```
`piclone` → GUI version included in Raspberry Pi OS (on Desktop edition, under Accessories → SD Card Copier).
`fsarchiver` or `partclone` → good for compressed filesystem snapshots instead of raw block copies.


## 4. Cloud or Git-based Backups (for projects)
- Code/config → push to GitHub.
- Data/logs → use `rclone` to sync with Google Drive/Dropbox/etc.


## 5. Hybrid Strategy
- Full SD card image once in a while (major upgrade, working state).
- Incremental `rsync` nightly for data/projects.
- That way if your SD card corrupts, you reflash the last full image and then overlay your recent project files.



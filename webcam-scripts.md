# Dahua webcam grabber and running-hour timelapse

On my vm, a full hour timelapse in ffmpeg takes much more CPU activity (in % and in time) rather than stitching 1min timelapses every 1min and then concatenating them together.

## /root/script.sh

a very dirty hack / script to run cron-powered grabbing every 5 seconds:

```bash
(sleep 5 && sh /root/ffmpeg_dahua.sh) &
(sleep 10 && sh /root/ffmpeg_dahua.sh) &
(sleep 15 && sh /root/ffmpeg_dahua.sh) &
(sleep 20 && sh /root/ffmpeg_dahua.sh) &
(sleep 25 && sh /root/ffmpeg_dahua.sh) &
(sleep 30 && sh /root/ffmpeg_dahua.sh) &
(sleep 35 && sh /root/ffmpeg_dahua.sh) &
(sleep 40 && sh /root/ffmpeg_dahua.sh) &
(sleep 45 && sh /root/ffmpeg_dahua.sh) &
(sleep 50 && sh /root/ffmpeg_dahua.sh) &
(sleep 55 && sh /root/ffmpeg_dahua.sh) &
(sleep 60 && sh /root/ffmpeg_dahua.sh) &
```

## /root/ffmpeg_dahua.sh

```console
/usr/bin/ffmpeg -loglevel debug -rtsp_transport tcp -i "rtsp://admin:pass@10.0.8.2:554/cam/realmonitor?channel=1&subtype=0" -vframes 1 /var/www/grab.jpeg -y
/usr/bin/cp -a /var/www/grab.jpeg "/var/www/grab-$(date +"%Y-%m-%d-%H-%M-%S").jpeg"
```

## /var/www/TL.sh

Timelapsing script:

```bash
/usr/bin/mkdir /var/www/timelapse-min
/usr/bin/mkdir /var/www/timelapse-vid
datetime=$(/usr/bin/date +%Y-%m-%d-%H-%M-%S)
datetimm=$(/usr/bin/date +%Y-%m-%d-%H-%M)
/usr/bin/find /var/www/grab-*.jpeg -maxdepth 1 -mmin -1 -type f -exec /usr/bin/cp "{}" /var/www/timelapse-min \;
/usr/bin/find /var/www/timelapse-min/ -name "*.jpeg" -printf "file '%p'\n" | /usr/bin/sort > /var/www/timelapse-min/list
/usr/bin/ffmpeg -f concat -safe 0 -i /var/www/timelapse-min/list -c copy -r 24 -vcodec libx264 -y /var/www/timelapse-min/timelapsenew.mp4
/usr/bin/mv /var/www/timelapse-min/timelapsenew.mp4 /var/www/timelapse-vid/tl-$datetimm.mp4
/usr/bin/rm -f /var/www/timelapse-min/*
/usr/bin/find /var/www/timelapse-vid/ -name "*.mp4" -mmin -60 -printf "file '%p'\n" | /usr/bin/sort > /var/www/timelapse-vid/list
/usr/bin/ffmpeg -f concat -safe 0 -i /var/www/timelapse-vid/list -c copy /var/www/timelapse.mp4 -y
```

Latest grabbed file and timestamped older files get stored directly in `/var/www`

Temporarily copied jpegs for latest-min timelapses get copied to `/var/www/timelapse-min`

Created mp4s get copied to `/var/www/timelapse-vid`

Running hour timelapse gets concatenated in `/var/www/timelapse.mp4`

## Crontab file


```console
* * * * * sh /root/script.sh > /dev/null
* * * * * sh /var/www/TL.sh > /dev/null
* * * * * /usr/bin/find /var/www -name "*.jpeg" -mmin +70 -exec /usr/bin/mv {} /mnt/archives/archives/archives-jpeg \;
* * * * * /usr/bin/find /var/www/timelapse-vid -name "*.mp4" -mmin +70 -exec /usr/bin/mv {} /mnt/archives/archives/archives-vid \;
```

Archived JPEGs and running-minute mp4s are copied to `/mnt/archives`, (which is a NAS drive) to keep vm drive clean.

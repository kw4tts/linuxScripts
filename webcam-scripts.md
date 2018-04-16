# Dahua webcam grabber and running-hour timelapse

On my vm, a full hour timelapse in ffmpeg takes much more CPU activity (in % and in time) rather than stitching 1min timelapses every 1min and then concatenating them together.

## /root/script.sh

a very dirty hack / script to run cron-powered grabbing every 5 seconds:

```console
(sleep 5 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 10 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 15 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 20 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 25 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 30 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 35 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 40 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 45 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 50 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 55 && sh /root/ffmpeg_dahua.sh > /dev/null) &
(sleep 60 && sh /root/ffmpeg_dahua.sh > /dev/null) &
```

## /root/ffmpeg_dahua.sh

```console
/usr/bin/ffmpeg -loglevel panic -rtsp_transport tcp -i "rtsp://admin:unior941337@172.30.10.237:554/cam/realmonitor?channel=1&subtype=0" -vframes 1 /var/www/grab.jpeg -y
/usr/bin/cp -a /var/www/grab.jpeg "/var/www/grab-$(date +"%Y-%m-%d-%H-%M-%S").jpeg" > /dev/null
```

## /var/www/TL.sh

Timelapsing script:

```console
#
# TL SCRIPT
#

#vars-paths
path_root=/var/www
path_jpeg_min=${path_root}/tl-min-jpeg
path_min=$path_root/tl-min
path_hr=$path_root/tl-hr
path_tl=$path_root/timelapse.mp4
path_tlnew=$path_root/timelapse.new.mp4

#vars
datetime=$(/usr/bin/date +%Y-%m-%d-%H-%M-%S)
datetimm=$(/usr/bin/date +%Y-%m-%d-%H-%M)
datetimm_prev=$(/usr/bin/date +%Y-%m-%d-%H-%M -d '-1 minute')
datetimh=$(/usr/bin/date +%Y-%m-%d-%H)
datetimh_prev=$(/usr/bin/date +%Y-%m-%d-%H -d '-1 hour')
mins=$(/usr/bin/date +%M)

mkdir -p $path_jpeg_min $path_min $path_hr

#1min TL - from PREV to now - named as PREV
/usr/bin/find $path_root -maxdepth 1 -name "grab-$datetimm_prev-*.jpeg" -type f -exec /usr/bin/cp "{}" $path_jpeg_min \;
/usr/bin/find $path_jpeg_min -maxdepth 1 -name "*.jpeg" -printf "file '%p'\n" | /usr/bin/sort > $path_jpeg_min/list
/usr/bin/ffmpeg -f concat -safe 0 -i $path_jpeg_min/list -c copy -r 24 -vcodec libx264 -y $path_min/tl-$datetimm_prev.mp4 -loglevel panic
/usr/bin/rm -f $path_jpeg_min/*

#1hr TL - from PREV to now - named as PREV
/usr/bin/find $path_min -maxdepth 1 -mmin -59 -name '*.mp4' -printf "file '%p'\n" | /usr/bin/sort > $path_tl_min/list
/usr/bin/ffmpeg -f concat -safe 0 -i $path_tl_min/list -c copy $path_tlnew -y -loglevel panic
/usr/bin/rm $path_tl_min/list -f
/usr/bin/mv $path_tlnew $path_tl


#archive TL
if [ $mins = "00" ]
then
  /usr/bin/find $path_min -maxdepth 1 -name "tl-$datetimh_prev-*.mp4" -printf "file '%p'\n" | /usr/bin/sort > $path_tl_min/list_hr
  /usr/bin/ffmpeg -f concat -safe 0 -i $path_tl_min/list_hr -c copy $path_hr/tl-$datetimh_prev.mp4 -y -loglevel panic
  /usr/bin/rm -f $path_tl_min/list_hr
fi

```

Latest grabbed file and timestamped older files get stored directly in `/var/www`

Temporarily copied jpegs for latest-min timelapses get copied to `/var/www/tl-min-jpeg`

Created mp4s get copied to `/var/www/tl-min`

Last hour timelapse gets concatenated in `/var/www/tl-hr/timelapse.mp4`

Running hour timelapse gets concatenated in `/var/www/timelapse.mp4`

## /root/crontab.sh

```console
echo "TLScript: started!"
/usr/bin/sh /root/script.sh > /dev/null
echo "TLScript: Snaps taken!"

/usr/bin/sh /var/www/TL.sh > /dev/null
echo "TLScript: Timelapse built!"

/usr/bin/find /var/www -name "*.jpeg" -mmin +70 -exec /usr/bin/mv {} /mnt/archives/archives/jpegs \;
/usr/bin/find /var/www/tl-min -name "*.mp4" -mmin +70 -exec /usr/bin/mv {} /mnt/archives/archives/tl-min \;
/usr/bin/find /var/www/tl-hr -name "*.mp4" -mmin +10 -exec /usr/bin/mv {} /mnt/archives/archives/tl-hr \;
#echo "TLScript: cleaned up!"
```

Archived JPEGs and running-minute mp4s are copied to `/mnt/archives`, (which is a NAS drive) to keep vm drive clean.


## Crontab file

```console
* * * * * sh /root/cronscript.sh
```

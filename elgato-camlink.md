Elgato Cam Link 4K and Microsoft Teams
--------------------------------------

Issues on Linux:

- Device claims capability of multiple pixel formats, but actually supports only one: https://assortedhackery.com/patching-cam-link-to-play-nicer-on-linux/
- Microsoft Teams only supports up top 1280x720 video: https://microsoftteams.uservoice.com/forums/908686-bug-reports/suggestions/42405061-linux-application-shows-black-video-on-webcam-with

Workaround

- use v4l2loopback
- rescale using ffmpeg, e.g.

````
ffmpeg -f v4l2 -input_format yuyv422 -video_size 1920x1080 -i /dev/video0 -codec rawvideo -vf scale=1280:720 -s 1280x720 -f v4l2 /dev/video2
````

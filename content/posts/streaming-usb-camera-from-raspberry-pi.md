---
# Title slug: streaming-usb-camera-from-raspberry-pi.md
title: "Streaming a USB camera from a Raspberry Pi (4B)"
date: 2023-08-20T16:52:00+02:00
# weight: 1
# aliases: ["/first"]
tags: ["utility"]
author: "Rick Lamers"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "How to stream a USB camera from a Raspberry Pi, tackling issues one by one."
canonicalURL: "https://ricklamers.io/posts/streaming-usb-camera-from-raspberry-pi"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

## A neat use case for Raspberry Pi

I was looking for a way to monitor my puppy's dog crate and I figured a good approach would be to use my PS3 eye USB camera with my Raspberry Pi 4 to stream audio and video to my phone. Buying a dedicated monitoring handheld felt like a waste of money, since I already had the hardware to do it, _in theory_. Of course, in practice you run into issues and in this article I'll describe the issues I ran into and how I solved them.

For most of the configuration and discovery I queried ChatGPT's `GPT-4` and it was surprisingly helpful. I was able to get a fully working solution within two hours.

## Firefighting issues

As the structure of the article I'll provide a play-by-play of the issues I encountered and how I solved them.

### Issue 1: Formatting Raspberry Pi for headless use

There was a [change](https://www.raspberrypi.com/news/raspberry-pi-bullseye-update-april-2022/) to how Raspberry Pi's with Raspberry Pi OS are configured for headless use. You have to create a user through the flashing utility as the default `pi` user is no longer available.

![Configure a user in the Raspberry Pi OS flasher utility](/headless-config.png)

In addition the flasher utility will create a `wpa_supplicant.conf` file in the boot partition of the SD card. This file contains the WiFi credentials for the Raspberry Pi to connect to the network. It also enables SSH if you configure it to do so.

### Issue 2: Choosing the streaming protocol

I wanted to stream the video and audio to my iPhone, so I needed a protocol that is supported by the available media apps. There are multiple options here: RTMP, RTSP, HLS, UDP based MPEG-TS, and WebRTC.

#### RTSP
Creating an RTSP stream that combined audio and video seemed rather troublesome because I kept being recommended by ChatGPT to independently stream the audio and video stream on different ports and then combine them on the client side. I didn't want to do that, so I explored the other options.

#### HLS
HLS usually has fairly high latency and my goal was to get within approximately 1 seconds of latency.

#### UDP based MPEG-TS
UDP based MPEG-TS required knowing the target IP address of the client, which I didn't want to do. I wanted to be able to connect to the stream from any device and multiple devices at the same time. In addition, UDP being a lossy protocol, I didn't want to deal with flakiness due to packet loss.

#### WebRTC
WebRTC seemed to cause difficulty on the client side of things. None of the streaming media apps I surveyed in the iOS App Store supported WebRTC.

#### RTMP
RTMP seemed to be the best option. It's supported by a lot of media apps and it's fairly easy to set up. I decided to go with RTMP.

### Issue 3: Setting up a streaming server
One thing I learned is that RTMP requires a server to be running on the Raspberry Pi. I decided to use `nginx-rtmp-module` as that was recommended in multiple places.

Install `nginx`. On Raspberry Pi OS Bullseye, `nginx` is available in the default repositories.

```sudo apt install nginx libnginx-mod-rtmp```

Configure the server to use the RTMP module by adding this block to the default configuration file `/etc/nginx/nginx.conf`:

```nginx
rtmp {
    server {
        listen 1935;
        chunk_size 4096;
        
        application live {
            live on;
            record off;
        }
    }
}
```

### Issue 4: 7 seconds of latency! Hardware acceleration for video encoding to the rescue
The `ffmpeg` command I was recommended by ChatGPT didn't seem to use the right codec. As a result the encoding was fully done by the CPU and I learned that it was the cause of significant latency, it added about 6 seconds of latency to the stream resulting in a total of 7 seconds of latency.

`ffmpeg ... -codec:v h264_omx ...`

I found out through a [GitHub issue](https://github.com/raspberrypi/firmware/issues/1366) that the recommended way to get `ffmpeg` to use hardware decoding on Raspberry Pi is to use the `-codec:v` encoder of `h264_v4l2m2m`.

### Issue 5: Fusing the microphone inputs
This issue is a bit particular to the PS3 eye USB camera. Its microphone has 4 channels. To fuse them to a stereo channel, I used the `pan` filter of `ffmpeg`. The relevant bit in the `ffmpeg` command is `-af pan=stereo|FL=0.5*FC+0.5*FL+0.5*BL|FR=0.5*FC+0.5*FR+0.5*BR`.

### Issue 6: Getting the services to start on boot
NGINX was easy: `sudo systemctl enable nginx`

For the `ffmpeg` command I created a systemd service file in `/etc/systemd/system/ffmpeg_stream.service``:

```ini
[Unit]
Description=FFmpeg Streamer
After=network.target

[Service]
ExecStart=/bin/bash -c 'while true; do ffmpeg -thread_queue_size 512 -s 640x480 -i /dev/video0 -thread_queue_size 1024 -f alsa -ac 4 -i hw:3,0 -codec:v h264_v4l2m2m -b:v 8096k -af "pan=stereo|FL=0.5*FC+0.5*FL+0.5*BL|FR=0.5*FC+0.5*FR+0.5*BR" -c:a aac -b:a 128k -r 30 -f flv rtmp://192.168.178.30/live/mystream || continue; done'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Issue 7: Audio disappearing after a while
For some reason after a few hours of streaming the audio would disappear. I'm not sure why this happens, but I found a workaround by restarting the `ffmpeg` service every hour. I knew debugging this issue would potentially take a long time, as every idea I had would require a few hours of testing. I decided to go with the workaround for now.

For the restart I created a dedicated systemd timer in `/etc/systemd/system/ffmpeg_stream-restart.timer`:

```ini
[Unit]
Description=Restart ffmpeg_stream every hour

[Timer]
OnBootSec=1h
OnUnitActiveSec=1h
Unit=ffmpeg_stream.service

[Install]
WantedBy=timers.target
```

### Issue 8: Finding the right iOS app
For some reason VLC refused to play my stream while VLC on macOS worked fine. I tried a few other apps and found that [EasyPlayerRTMP](https://apps.apple.com/nl/app/easyplayerrtmp/id1347047886?l=en-GB) worked fine.


### Issue 9: Low-light camera performance
While the framerate (it can easily do 60 fps and can even go up to 120 fps) and audio quality capabilities of the PS3 Eye are excellent the low-light performance is not. For low-light situations a different camera would be better. I explored several options but landed on [ArduCam B0205](https://www.arducam.com/product/b0205-arducam-1080p-day-night-vision-usb-camera-module-for-computer-2mp-automatic-ir-cut-switching-all-day-image-usb2-0-webcam-board-with-ir-leds/) that features a motorized IR-CUT filter and infrared LEDs. It's USB based so should work as a drop-in replacement for the PS3 Eye. I've not yet received the camera, but I'll update this article once I've tested it.

## Look ma, it works!
In the end I was able to get a working stream with about 1 second of latency which is good enough for my use case.

![Stream working](/stream-working.jpg)
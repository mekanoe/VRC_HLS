# VRChat Streaming Guide
Solution &amp; guide for streaming anything into VRChat via HLS or other similar technologies that can be consumed by AVPro in Unity.

# Synopsis

We can use nginx's RTMP module to output a usable stream for VRChat video sync panels. This provides a very good quality, low latency, uninterruptable stream so you can watch whatever you'd like, much like Rabbit. Some future work on this repo would be making a Dockerized version of this solution, a mass of helper systems, and a paid web service to allow it's hosted usage too.

Some of this work has already been done and [documented before in Japanese](https://github.com/yukimochi/VRC_HLS), but this repo is my own work and information on how to do it without anything more complicated.

## Technical Specifications

### Requirements (Basic Setup)

- web-accessible server or PC, cloud or locally hosted OK, behind CloudFlare also OK (and possibly recommended.)
- nginx with RTMP module
  + `$ apt install nginx libnginx-mod-rtmp`
- OBS or other RTMP output capable software (ffmpeg, wirecast, XSplit...)
- A VRChat world that accept livestreams ([Bedroom Theater](https://www.vrchat.net/home/launch?worldId=wrld_fff5d510-fc53-4e88-9d4e-1e0e45a17aff~friends))
- (Optionally VLC to test HLS)

This can be done all in one PC, just set up port-forwarding.

#### nginx configuration

in `nginx.conf` (or as an include but NOT sites-enabled,) add:

```
rtmp {
  server {
    listen 1935; # this is the port, you may change it if necessary.
    chunk_size 4000; # a basic chunk_size for RTMP's uses.
    
    application live { # "live" may be changed to whatever you'd like, this will affect the URLs we use later, though.
      live on;
      
      allow publish 127.0.0.1 192.168.0.1/1*6; # this may include your entire network, you may be more or less exact with this. 
                                               # if you don't care about potential griefing, use `all' in place of the IPs.
      deny publish all; # denied if none of the above apply.
      
      # -- HLS --
      hls on;
      hls_path /tmp/hls; # this is where all of HLS's files will be stored. 
                         # They are a running replacement so disk space will be minimal. 
                         # Adjust other HLS settings if necessary to level it out if it's a problem.
      
      # optionally,
      #hls_continuous on;
      # Uncommenting will fix a minor issue if the stream ends prematurely.
      # However... this does have a side effect of using more disk space.
    }
  }
}
```

somewhere in an `http` block, or sites-enabled...

```
server {
  listen 80;
  server_name vrcstream.example.com; // replace or remove this value as needed.
   
  # setting up any HTTP configuration you'd like is ok, the only important part is below.
  
  location /hls {
    # Serve HLS fragments
    types {
        application/vnd.apple.mpegurl m3u8;
        video/mp2t ts;
    }
    root /tmp;
    add_header Cache-Control no-cache;
  }
}
```

#### OBS configuration (or other RTMP)

This is considering you have stuck with the original application directive above in nginx.

We will use an RTMP URL `rtmp://vrcstream.example.com/live/random-stream-key`. Take note of the end part, as this will be considered in the HLS URL later.

For OBS, we'd set this up in the form of
![OBS example](https://pomf.pyonpyon.moe/dfuufi.png)

1. Use **Custom Streaming Server**
2. Use the beginning part of the RTMP URL in **URL**, `rtmp://vrcstream.example.com/live`
3. Use the end part as the **Stream Key**, `random-stream-key` (this value can be anything, but take note of it.)

#### Testing in VRChat (or VLC)

For our example, the HLS playlist URL would be `http://vrcstream.exmaple.com/hls/random-stream-key.m3u8`

For VRChat, Enter a world that can accept livestreams (possibly videos in general,) type in the above URL (I'd recommend typing it in notepad and pasting if you can), then click play, and enjoy!

For VLC, go to **File**, **Network Stream**, and enter in the above URL, and Play!

### Quirks of nginx RTMP HLS method

- There is about a 60 second delay with this configuration. I will research how to get this time down.
- It will never desync for other players so you may stop/pause/start to your heart's content, but it will start from the beginning of the buffer.
- VRChat can break state sync, and sometimes end up draining out the 60 second buffer.

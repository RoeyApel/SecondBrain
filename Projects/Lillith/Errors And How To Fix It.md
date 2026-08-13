### logs in db:
`filewatch image:in-a-box_db: Windows I/O overflow.` `You may be able to fix this by setting the env var TILT_WATCH_WINDOWS_BUFFER_SIZE.` `Current buffer size: 65536` `More details: [https://github.com/tilt-dev/tilt/issues/3556](https://github.com/tilt-dev/tilt/issues/3556)` `Caused by: short read in readEvents()` `filewatch image:in-a-box_db: Windows I/O overflow.` `You may be able to fix this by setting the env var TILT_WATCH_WINDOWS_BUFFER_SIZE.` `Current buffer size: 65536` `More details: [https://github.com/tilt-dev/tilt/issues/3556](https://github.com/tilt-dev/tilt/issues/3556)` `Caused by: short read in readEvents()` `filewatch image:in-a-box_db: Windows I/O overflow.` `You may be able to fix this by setting the env var TILT_WATCH_WINDOWS_BUFFER_SIZE.` `Current buffer size: 65536` `More details: [https://github.com/tilt-dev/tilt/issues/3556](https://github.com/tilt-dev/tilt/issues/3556)`

### fix: retrigger the db

------
###  logs in video service:
`video: video/camera/4a81b36b-a634-4d34-b9b4-fcfeeb6fe875/2026/08/13/11/1786621994240_0.ts, status: fail, stored_at: db` `2026/08/13 11:53:38 ERR [path e4ca3bde-6609-4005-ad52-52bf76f117fb] [RTSP source] bad status code: 401 (Unauthorized)`
### fix: 
change camera url to have init the password and the username like that:
```
rtsp://admin:Maccabi14@10.0.10.34/Streaming/Channels/101?transportmode=unicast&profile=Profile_1
```
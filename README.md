# How to Record Livestream from OSEE GoStream directly to PC without OBS

## Installation 

Install Prerequisites:
```powershell
winget Git.Git Docker.DockerDesktop -h
```

```powershell
git clone github.com/maniacboy777/rtmp-server && cd rtmp-server && docker compose up -d
```


Open `GoStream Control` software on PC

Click `Stream > Create Stream XML File`

For `Platform Name`you can put whatever you like such as `PC Recording`

For `URL` put `rtmp://your-computers-ip-address:1935/live`
Note: If you don't know your IP address you can easily find it by typing `ipconfig` into cmd or terminal

Click `EXPORT FILE`

Click `Stream > Open Streaming Dialogue`

Select one of the 3 available streams

Under `Platform` you should see the Platform you just created. Select it.

`Stream Key` should be whatever you want the subfolder to be called under Videos such `livestream`

Ensure `Enable Live` is checked

Now when you go live, it should automatically stream straight to an MP4 file in `Videos/livestream` or whatever you specified as a stream key.
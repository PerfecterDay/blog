# ffmpeg 
{docsify-updated}

## 分割视频文件为几个小文件
```
ffmpeg -i app-ttl.mp4 -acodec copy -f segment -segment_time 1200 -vcodec copy -reset_timestamps 1 -map 0 app-ttl-%d.mp4
```

分割  app-ttl.mp4 为多个文件，每个文件 1200s（20分钟）时长。

## 下载视频
ffmpeg -i 'xxxxxxxx.m3u8' test.mp4

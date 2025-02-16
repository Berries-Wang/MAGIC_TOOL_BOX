# [FFmpeg](https://ffmpeg.org/ffmpeg.html#Video-Options)
A complete, cross-platform solution to record, convert and stream audio and video.（用于录制、转换和传输音频和视频的完整跨平台解决方案。）

FFmpeg 是一个功能强大且开源的多媒体框架，几乎可以处理任何音频或视频格式。它广泛用于处理多媒体文件和流媒体，并支持大量的编解码器和格式。

## 使用案例
### 1. 将视频按时长切分
```shell
 ../ffmpeg -i 81e0f05d8fe9dd87_mp4_355043993582_mp4_264_hd_taobao.mp4 -c copy -map 0  -segment_time 1800 -f segment output%03d.mp4 
  -c copy表示使用“复制”模式，不重新编码视频，以保持视频质量；
  -map 0表示映射所有输入流（视频、音频等）；
  -segment_time 1800指定每个分割段的时间长度为1800秒（即30分钟）；
  -f segment指定分割输出格式；output%03d.mp4表示输出文件名格式，其中%03d表示用三位数字表示序号，如output001.mp4、output002.mp4等
 ‌     文件前缀必须是output ， 否则程序不执行
```

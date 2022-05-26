# video-super-resolution-by-realesrgan
Using realesrgan to achieve video super-resolution.

使用 realesrgan 和 ffmpeg 对视频进行超分辨率。

## 思路

1. 使用 ffmpeg 对源视频中的视频轨道进行抽帧。这一步将会得到一个文件夹，为抽帧的图片序列。
2. 使用 realesrgan 对抽帧的结果序列进行超分辨率处理，得到另一个文件夹，里面为已经超分辨率的图片序列。
3. 使用 ffmpeg 将已经超分辨率的图片序列和源视频音轨进行合并。

> 注意：抽帧和合并过程中，假设的视频帧率应当和源视频保持几乎一致。举例，源视频为24fps，那么抽帧按24fps提取，合并也按照24fps合并。

## 详细的步骤
下面以一个视频文件 the-last-moment@Lightpi.flv 作为示例进行说明。

### ffmpeg 掌握源视频的必要信息
```
ffmpeg -i .\the-last-moment@Lightpi.flv
```
关键输出信息如下：
```
Input #0, flv, from '.\the-last-moment@Lightpi.flv':
  Metadata:
    description     : Bilibili VXCode Swarm Transcoder r0.2.61(gap_fixed:False)
    metadatacreator : Version 1.9
    hasKeyframes    : true
    hasVideo        : true
    hasAudio        : true
    hasMetadata     : true
    canSeekToEnd    : true
    datasize        : 19331857
    videosize       : 17054253
    audiosize       : 2245688
    lasttimestamp   : 109
    lastkeyframetimestamp: 109
    lastkeyframelocation: 19332933
  Duration: 00:01:49.28, start: 0.100000, bitrate: 1415 kb/s
  Stream #0:0: Video: h264 (High), yuv420p(tv, bt470bg/unknown/unknown, progressive), 960x540, 1248 kb/s, 30.03 fps, 30 tbr, 1k tbn
  Stream #0:1: Audio: aac (LC), 44100 Hz, stereo, fltp, 160 kb/s
```
在这里，关注两个数值：源视频的时长和帧率。
- Duration: 00:01:49.28
- 30.03 fps 
 
我们希望预计将来抽帧得到的图片序列是多少张? 计算公式:
```
图片序列数 = 视频时长（秒数s）x 帧率（fps）
```
因此，可以计算得到 109 x 30.03 = 3273。这里，舍掉了视频时长中的 .28s。

### ffmpeg 抽帧
```
ffmpeg -i the-last-moment@Lightpi.flv -qscale:v 1 -qmin 1 -qmax 1 -vsync 0 tmp_frames/frame%08d.jpg
```
- -qscale:v 1 ：保持视频尺寸缩放比例不变。
- -qmin 1 -qmax 1 ：保持缩放质量不变，此时最小和最大都设置为1，结果就是1。
- -vsync 0 ：每一帧都从demuxer到muxer，保持原样。 [vsync](https://ffmpeg.org/ffmpeg.html#toc-Advanced-options)

这一步，得到3273张图片原帧。

### realesrgan 超分
```
./realesrgan-ncnn-vulkan.exe -i tmp_frames -o out_frames -n realesr-animevideov3 -s 2 -f jpg
```
- -i tmp_frames 输入文件夹
- -o out_frames 输出文件夹
- -n realesr-animevideov3 指定的算法模型
- -s 2 缩放比例，这里是x2倍。可选数值取决于算法模型支持的枚举值，以及使用者期望的缩放值。
- -f jpg 输出的图片文件格式。这里指定为jpg。

等待超分结束后，得到3273张图片超分辨率帧。**根据视频超分辨率的参数设置以及显卡算力的强弱，运行时长会有明显的差异。**

### ffmpeg 合并
```
ffmpeg -framerate 30.03 -i out_frames/frame%08d.jpg -i .\the-last-moment@Lightpi.flv 
-map 0:v:0 -map 1:a:0 -c:a copy -c:v libx264 -r 30.03 -pix_fmt yuv420p -t 00:01:49.00 output_w_audio_4.flv
```
这个命令比较长，参数说明如下：
> 参数文档参考 [-map [-]input_file_id[:stream_specifier][?][,sync_file_id[:stream_specifier]] | [linklabel] (output)](
  https://ffmpeg.org/ffmpeg.html#toc-Advanced-options)
- -framerate 30.03 ：指定读取的帧率，这里按照30.03帧率读取。
- -i out_frames/frame%08d.jpg ：指定一个输入流，这里为超分后的文件夹。这是我们需要的视频流。
- -i .\the-last-moment@Lightpi.flv：指定另一个输入流，这里为源视频。我们需要在后续获取并处理这个音频轨道。
- `-map 0:v:0` map处理的是流映射关系，这里 `0:v:0`中第一个0表示索引为0输入流，即 out_frames ；中间的v表示流修饰符，表示只对视频流进行处理；末尾的0表示输出流的索引，指的是下文的输出文件，output_w_audio_4.flv。
- `-map 1:a:0` 根据上面的解释，这里是对 `-i .\the-last-moment@Lightpi.flv `取音频流，输出到最后的输出文件 output_w_audio_4.flv。
- -c:a copy 对音频流的处理。-c含义是codec, :a 表示处理音频, copy表示仅复制。
- -c:v libx264 对视频流的处理。-c含义是codec, :v 表示处理视频, libx264为编码器。
- -r 30.03 指定输出文件的帧率，这里设置为和输入帧率一致。
- -pix_fmt yuv420p 指定颜色编码方法，这里是YUV4:2:0。传值yuv420p。
- -t 00:01:49.00 指定输出文件的时长，这里以 109 x 30.03 = 3273 这个计算为基准。
- output_w_audio_4.flv 输出的文件名称。

## Reference
- [YUV](https://zh.m.wikipedia.org/zh/YUV)
- https://github.com/xinntao/Real-ESRGAN/blob/master/docs/anime_video_model.md

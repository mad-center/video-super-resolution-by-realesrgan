# video-super-resolution-by-realesrgan
Using realesrgan to achieve video super-resolution.

使用 realesrgan 和 ffmpeg 对视频进行超分辨率。

## 思路

1. 使用 ffmpeg 对源视频中的视频轨道进行抽帧。这一步将会得到一个文件夹，为抽帧的图片序列。
2. 使用 realesrgan 对抽帧的结果序列进行超分辨率处理，得到另一个文件夹，里面为已经超分辨率的图片序列。
3. 使用 ffmpeg 将已经超分辨率的图片序列和源视频音轨进行合并。

> 注意：抽帧和合并过程中，假设的视频帧率应当和源视频保持几乎一致。举例，源视频为24fps，那么抽帧按24fps提取，合并也按照24fps合并。

## 详细的步骤

### 掌握源视频的必要信息

### ffmpeg 抽帧
```
ffmpeg -i the-last-moment@Lightpi.flv -qscale:v 1 -qmin 1 -qmax 1 -vsync 0 tmp_frames/frame%08d.jpg
```
- -qscale:v 1 ：保持视频尺寸缩放比例不变。
- -qmin 1 -qmax 1 ：保持缩放质量不变，此时最小和最大都设置为1，结果就是1。
- -vsync 0 ：每一帧都从demuxer到muxer，保持原样。 [vsync](https://ffmpeg.org/ffmpeg.html#toc-Advanced-options)

### realesrgan 超分
```
./realesrgan-ncnn-vulkan.exe -i tmp_frames -o out_frames -n realesr-animevideov3 -s 2 -f jpg
```
- -i tmp_frames 输入文件夹
- -o out_frames 输出文件夹
- -n realesr-animevideov3 指定的算法模型
- -s 2 缩放比例，这里是x2倍。可选数值取决于算法模型支持的枚举值，以及使用者期望的缩放值。
- -f jpg 输出的图片文件格式。这里指定为jpg。

等待超分结束后，

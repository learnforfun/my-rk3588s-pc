# 基于ROC-RK3588S-PC做一个DASHCAM

## 任务一 mipi摄像头模组OS08A20调试
基于官方驱动，dts要自己调试一下，后续再完善文档。
要点
dts要改
首次编译，要把rootfs下载回来
首次刷机，要构建完整的镜像

## 任务二 USB3.0摄像头调试
即插即用。
调试命令
* 实时预览
```
# 1. 确保挂载本地桌面显示变量
export DISPLAY=:0

# 2. 启动预览管道
gst-launch-1.0 v4l2src device=/dev/video20 ! \
image/jpeg,width=1920,height=1080,framerate=60/1 ! \
mppjpegdec ! \
xvimagesink

```
* 录像保存(有横条纹)
```
gst-launch-1.0 v4l2src device=/dev/video20 num-buffers=250 ! video/x-raw,format=YUY2,width=1920,height=1080,framerate=50/1 ! mpph265enc ! h265parse ! mp4mux ! filesink location=~/Desktop/yuyv_50fps.mp4
```
编译gstreamer-rga后使用下面的命令，保存的视频很好。
```
gst-launch-1.0 -e v4l2src device=/dev/video20 ! 'video/x-raw, format=YUY2, width=2560, height=1440, framerate=30/1' ! videoconvert ! 'video/x-raw, format=BGR, width=2560, height=1440, framerate=30/1' ! queue ! rgavideoconvert core-mask=rga3 ! 'video/x-raw, format=NV12, width=2560, height=1440, framerate=30/1' ! mpph265enc bps=10000000 rc-mode=cbr ! h265parse ! mp4mux ! filesink location=raw_rga3_bgr_test.mp4
```



目前确认的结果：

摄像头直接录 YUY2：正常，无横条纹。
videoconvert 软件转换 YUY2 → NV12：正常，无横条纹。
RGA 默认采集路径：有横条纹；强制 core-mask=rga3 仍有条纹。
RGA 配合 io-mode=mmap、core-mask=rga3 转成 NV12：已验证正常。
io-mode=rw 因摄像头驱动不支持而失败。
已验证正常：录制 RGA 转换后的 NV12 raw
gst-launch-1.0 -e v4l2src device=/dev/video20 io-mode=mmap num-buffers=300 \
  ! video/x-raw,format=YUY2,width=1280,height=720,framerate=30/1 \
  ! rgavideoconvert core-mask=rga3 \
  ! video/x-raw,format=NV12,width=1280,height=720 \
  ! filesink location=mmap.nv12
播放查看：
ffplay -f rawvideo -pixel_format nv12 \
  -video_size 1280x720 -framerate 30 mmap.nv12

录制摄像头原始 YUY2
这条绕过 RGA，适合保存摄像头原始画面；你确认直接录制 YUY2 无条纹：
gst-launch-1.0 -e v4l2src device=/dev/video20 num-buffers=300 \
  ! video/x-raw,format=YUY2,width=1280,height=720,framerate=30/1 \
  ! filesink location=direct.yuy2

ffplay -f rawvideo -pixel_format yuyv422 \
  -video_size 1280x720 -framerate 30 direct.yuy2

录制 H.265
可在已验证正常的 mmap→RGA→NV12 路径后接硬件编码器：
gst-launch-1.0 -e v4l2src device=/dev/video20 io-mode=mmap num-buffers=300 \
  ! video/x-raw,format=YUY2,width=1280,height=720,framerate=30/1 \
  ! rgavideoconvert core-mask=rga3 \
  ! video/x-raw,format=NV12,width=1280,height=720 \
  ! mpph265enc \
  ! h265parse \
  ! filesink location=mmap.h265

播放查看：
ffplay -f hevc mmap.h265

**说明：**上面的 H.265 命令采用了已验证无条纹的 RGA 输入路径，但目前你验证的是 RGA 输出的 NV12 raw；这条 H.265 录制结果还需要实际跑一次确认。


拍照
```
mkdir -p raw-720p60

gst-launch-1.0 -e \
  v4l2src device=/dev/video20 io-mode=mmap num-buffers=10 \
  ! video/x-raw,format=YUY2,width=1280,height=720,framerate=60/1 \
  ! multifilesink location="raw-720p60/frame-%03d.yuy2"

ffplay -f rawvideo -pixel_format yuyv422 \
  -video_size 1280x720 raw-720p60/frame-000.yuy2
```

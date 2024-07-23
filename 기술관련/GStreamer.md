- 클라이언트(언리얼 엔진) 파이프라인 
  - RTSP
    rtspsrc location=rtsp://192.168.0.7:5000/zed-stream latency=0 ! decodebin ! videoconvert ! video/x-raw,format=(string)RGBA ! videoconvert ! appsink name=sink2 
  - UDP
    udpsrc port=5000 ! application/x-rtp,clock-rate=90000,payload=96 ! queue ! rtph264depay ! h264parse ! avdec_h264 ! queue ! videoconvert ! video/x-raw,format=(string)RGBA ! videoconvert ! appsink name=sink2 

  gst-launch-1.0 rtspsrc location=rtsp://192.168.0.12:5000/ ! rtph264depay ! h264parse ! avdec_h264 ! videoconvert ! autovideosink
  
- 서버 파이프 라인
  - RTSP
    gst-zed-rtsp-launch --address=192.168.0.12 --port=5000 zedsrc ! videoconvert ! 'video/x-raw, format=(string)I420' ! x264enc ! rtph264pay pt=96 name=pay0
  -  UDP
	gst-launch-1.0 zedsrc ! timeoverlay ! queue max-size-time=0 max-size-bytes=0 max-size-buffers=0 ! autovideoconvert ! x264enc byte-stream=true tune=zerolatency speed-preset=ultrafast bitrate=3000 ! h264parse ! rtph264pay config-interval=-1 pt=96 ! queue ! udpsink clients=192.168.0.2:5000 max-bitrate=3000000 sync=false async=false

gst-zed-rtsp-launch --address=192.168.0.12 --port=5000 zedsrc !  queue max-size-time=0 max-size-bytes=0 max-size-buffers=0 ! videoconvert ! x264enc byte-stream=true tune=zerolatency speed-preset=ultrafast bitrate=3000 ! h264parse ! rtph264pay config-interval=-1 pt=96 name=pay0


```
gst-launch-1.0 zedsrc ! timeoverlay ! tee name=split has-chain=true ! \
 queue ! autovideoconvert ! fpsdisplaysink \
 split. ! queue max-size-time=0 max-size-bytes=0 max-size-buffers=0 ! autovideoconvert ! \
 x264enc byte-stream=true tune=zerolatency speed-preset=ultrafast bitrate=3000 ! \
 h264parse ! rtph264pay config-interval=-1 pt=96 ! queue ! \
 udpsink clients=192.168.1.169:5000 max-bitrate=3000000 sync=false async=false
```
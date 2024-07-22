- 클라이언트(언리얼 엔진) 파이프라인 
  rtspsrc location=rtsp://192.168.0.7:5000/zed-stream ! decodebin ! videoconvert ! video/x-raw,format=(string)RGBA ! videoconvert ! appsink name=sink2 
  
- 클라이언트 파이프라인 
  gst-launch-1.0 rtspsrc location=rtsp://192.168.0.7:5000/zed-stream ! decodebin ! videoconvert ! video/x-raw,format=(string)RGBA ! videoconvert ! appsink name=sink2 
  
- 서버 파이프 라인
  gst-zed-rtsp-launch --address=192.168.0.12 --port=5000 zedsrc ! videoconvert ! 'video/x-raw, format=(string)I420' ! x264enc ! rtph264pay pt=96 name=pay0
   
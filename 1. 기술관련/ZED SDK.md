- ZED Camera 
- [[GStreamer]]로 활용하기
  Stereolaps 에서 Gstreamer Package를 제공해준다 이를 활용해서 비디오 정보를 넘겨준다. 

- ZED SDK UE Plugin 
  - 카메라 초기화 부분
    void USlCameraProxy::Internal_OpenCamera(const FSlInitParameters& InitParameters)
    이 부분에서 초기화 정보를 가지고 카메라를 실행하는 API를 실행해줌
  - 트래킹 초기화 부분
    ZEDCamera가 초기화 된 이후에 void AZEDPlayerController::ZedCameraOpened() 가 발생하고 설정에 따라 트래킹 설정을 초기화 하고 
  - 
- 
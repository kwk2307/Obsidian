
1. 실행
   - Open Level 시 Option으로 넣는 방식![](file:///C:/Users/VIVEST~1/AppData/Local/Temp/msohtmlclip1/01/clip_image002.png)
   - 월드 세팅에서 설정하는 방식  ![](file:///C:/Users/VIVEST~1/AppData/Local/Temp/msohtmlclip1/01/clip_image004.jpg)
2. 구성![](file:///C:/Users/VIVEST~1/AppData/Local/Temp/msohtmlclip1/01/clip_image006.png)
   - GameFeatures to Enable : 로드 후 실행할 GameFeature들
   - Default Pawn Data : 생성할 Pawn의 정보
     
3. Default Pawn Data  
   
4. 전반적인 흐름  
   LyraGameMode가 Init 될때 Lyra Experience Definition 를 초기화 하는 이벤트를 예약해 다음 프레임에 실행되게 한다.  
   GameFeatures to Enable 에 있는 GameFeature 로드 후 실행 -> Action Sets에 있는 GameFeatures 로드 후 실행 -> Actions 에 있는 Action 들 실행 -> Action Sets에 있는 Action 들 실행


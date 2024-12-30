## 개요

이 문서는 KAI POC2 프로젝트에서 사용된 기술과 해당 기술의 활용 방식을 정리한 문서입니다. 각 기술의 역할, 구현 방법, 및 코드 예제를 포함합니다.

---

## 1. 사용 기술 목록
    
- **멀티플레이어 네트워크 시스템** - **[[Unreal Engine Networking and Multiplayer]]** 를 활용한 온라인 기능 구현
    

---

## 2. 기술 상세 설명

### 2.1  멀티플레이어 네트워크 시스템

**설명:** 멀티플레이 기능을 구현하여 다수의 플레이어가 네트워크를 통해 상호작용.

**활용 방식:**

- 서버/클라이언트 구조 구축.
    
- 리플리케이션을 통한 데이터 동기화.

- RPC(Remote Procedure Calls) 사용.

**블루프린트 예제**

- 서버/클라이언트 구조 구축 
	**서버 - 세션 생성**
![[Pasted image 20241230152316.png]]
	**클라이언트-세션 검색**
![[Pasted image 20241230152544.png]]
	**클라이언트 - 세션 참가 
![[Pasted image 20241230152618.png]]
- 리플리케이션을 통한 데이터 동기화
	**PlayerController 정보 Replication** 관련 블루프린트 
![[Pasted image 20241230163049.png]]
	Color 변수의 디테일 화면. 리플리케이션 설정을 해줘야 한다. 
![[Pasted image 20241230163702.png]]
	**주의 사항**: 리플리케이션은 **서버에서만** 발생한다. 클라이언트의 변수를 리플리케이션 하려면 다른 방식을 사용해야 한다. 
	
-  RPC(Remote Procedure Calls) 사용.
	**화면 공유 요청** 관련 블루프린트 
![[Pasted image 20241230155311.png]]
	1. IETMWindowShareEvent : 클라이언트/서버에서 "화면공유" 버튼을 선택했을 시 발생하는 이벤트 Control_ShareEvent_ROS 이벤트를 발생 시킨다. 
	2. Control_ShareEvent_ROS : 서버에서만 실행되는 RPC Control_ShareEvent_MTC 이벤트를 발생시킨다. 
	   **주의 사항**: RunOnServer RPC의 경우 PlayerController가 **소유한 액터에서만 발생시킬** 수 있다.
	   위 경우 
	   1. Pawn의 Component 임 
	   2. Pawn은 PlayerContoller가 Pose 하고 있음
	   3. PlayerController가 Pawn을 소유하고 있음 
	   위와 같은 이유로 발생시킬 수 있다.
	3. Control_ShareEvent_MTC : 클라이언트/서버에 모두 발생하는 이벤트 팝업을 생성함

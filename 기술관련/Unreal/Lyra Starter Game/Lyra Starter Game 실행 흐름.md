참고자료
- https://lemonyun.tistory.com/147 
- https://velog.io/@1000/Lyra-%EB%9D%BC%EC%9D%B4%EB%9D%BC-%EC%8A%A4%ED%83%80%ED%84%B0-%EA%B2%8C%EC%9E%84-%EA%B0%9C%EC%9A%94-%EC%9A%94%EC%95%BD 
- https://velog.io/@woolzam/%EC%96%B8%EB%A6%AC%EC%96%BC-Lyra-Sample-Game-%EA%B2%8C%EC%9E%84-%ED%9D%90%EB%A6%84
**전반적인 흐름**

![[Pasted image 20240709150022.png]]GameState의 생성자에서 AbilitySystemComponent 과 ExperienceManagerComponent를 초기화 해줌 

![[Pasted image 20240709151006.png]]LyraGameMode의 InitGame 에서 LyraGameMode의 HandleMatchAssignmentIfNotExpectingOne 함수를 다음 틱에 실행 되도록 등록 하고 HandleMatchAssignmentIfNotExpectingOne 함수에서 현재 레벨의 Experience를 결정 해줌 

![[Pasted image 20240709165608.png]]
LyraGameMode 의 OnMatchAssignmentGiven 함수를 통해LyraExperienceManagerComponent의  SetCurrentExperience 함수를 호출함 

![[Pasted image 20240712152452.png]]
AssestManager 를 통해 ULyraExperienceDefinition 를 로드하고 CurrentExperience에 넣어준 후 StartExperienceLoad 함수 호출 






ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted() << 로드가 완료되면 여기서 Action을 하나씩 발생? 시킴


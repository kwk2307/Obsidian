https://lemonyun.tistory.com/147 <<참고

- 전반적인 흐름

GameState의 생성자에서 AbilitySystemComponent 과 ExperienceManagerComponent를 초기화 해줌 
![[Pasted image 20240709150022.png]]

LyraGameMode의 InitGame 에서 LyraGameMode의 HandleMatchAssignmentIfNotExpectingOne 함수를 다음 틱에 실행 되도록 등록 하고 HandleMatchAssignmentIfNotExpectingOne 함수에서 현재 레벨의 Experience를 결정 해줌 
![[Pasted image 20240709151006.png]]

LyraGameMode 의 OnMatchAssignmentGiven 함수를 통해LyraExperienceManagerComponent의  SetCurrentExperience 함수를 호출함 
![[Pasted image 20240709165608.png]]



ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted() << 로드가 완료되면 여기서 Action을 하나씩 발생? 시킴


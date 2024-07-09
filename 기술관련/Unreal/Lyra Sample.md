
- 전반적인 흐름

GameState의 생성자에서 AbilitySystemComponent 과 ExperienceManagerComponent를 초기화 해줌 
![[Pasted image 20240709150022.png]]


**LyraExperienceManagerComponent** << 여기서 Experience 처리
       UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(PluginURL, FGameFeaturePluginLoadComplete::CreateUObject(this, &ThisClass::OnGameFeaturePluginLoadComplete));

AssetManager 부분 확인해야함



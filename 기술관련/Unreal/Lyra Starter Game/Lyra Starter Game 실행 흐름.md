참고자료
- https://lemonyun.tistory.com/147 
- https://velog.io/@1000/Lyra-%EB%9D%BC%EC%9D%B4%EB%9D%BC-%EC%8A%A4%ED%83%80%ED%84%B0-%EA%B2%8C%EC%9E%84-%EA%B0%9C%EC%9A%94-%EC%9A%94%EC%95%BD 
- https://velog.io/@woolzam/%EC%96%B8%EB%A6%AC%EC%96%BC-Lyra-Sample-Game-%EA%B2%8C%EC%9E%84-%ED%9D%90%EB%A6%84
**전반적인 흐름**

*LyraExperienceDefinition*

![[Pasted image 20240709150022.png]]GameState의 생성자에서 AbilitySystemComponent 과 ExperienceManagerComponent를 초기화 해줌 

![[Pasted image 20240709151006.png]]LyraGameMode의 InitGame 에서 LyraGameMode의 HandleMatchAssignmentIfNotExpectingOne 함수를 다음 틱에 실행 되도록 등록 하고 HandleMatchAssignmentIfNotExpectingOne 함수에서 현재 레벨의 Experience를 결정 해줌 

![[Pasted image 20240709165608.png]]
LyraGameMode 의 OnMatchAssignmentGiven 함수를 통해LyraExperienceManagerComponent의  SetCurrentExperience 함수를 호출함 

![[Pasted image 20240712152452.png]]
AssestManager 를 통해 ULyraExperienceDefinition 를 로드하고 CurrentExperience에 넣어준 후 StartExperienceLoad 함수 호출 

```
	TSet<FPrimaryAssetId> BundleAssetList;
    TSet<FSoftObjectPath> RawAssetList;

    BundleAssetList.Add(CurrentExperience->GetPrimaryAssetId());

    for (const TObjectPtr<ULyraExperienceActionSet>& ActionSet : CurrentExperience->ActionSets)

    {

        if (ActionSet != nullptr)

        {

            BundleAssetList.Add(ActionSet->GetPrimaryAssetId());

        }

    }

  

    // Load assets associated with the experience

  

    TArray<FName> BundlesToLoad;

    BundlesToLoad.Add(FLyraBundles::Equipped);

  

    //@TODO: Centralize this client/server stuff into the LyraAssetManager

    const ENetMode OwnerNetMode = GetOwner()->GetNetMode();

    const bool bLoadClient = GIsEditor || (OwnerNetMode != NM_DedicatedServer);

    const bool bLoadServer = GIsEditor || (OwnerNetMode != NM_Client);

    if (bLoadClient)

    {

        BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateClient);

    }

    if (bLoadServer)

    {

        BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateServer);

    }

  

    TSharedPtr<FStreamableHandle> BundleLoadHandle = nullptr;

    if (BundleAssetList.Num() > 0)

    {

        BundleLoadHandle = AssetManager.ChangeBundleStateForPrimaryAssets(BundleAssetList.Array(), BundlesToLoad, {}, false, FStreamableDelegate(), FStreamableManager::AsyncLoadHighPriority);

    }
```
BundleAssetList 에 CurrentExperience와 ActionSet에 등록되어 있는 PrimaryAsset(ULyraExperienceDefinition, ULyraExperienceActionSet 등이 PrimaryAsset을 상속받음)의 ID를 가져와서 저장함

```
    //@TODO: Centralize this client/server stuff into the LyraAssetManager

    const ENetMode OwnerNetMode = GetOwner()->GetNetMode();

    const bool bLoadClient = GIsEditor || (OwnerNetMode != NM_DedicatedServer);

    const bool bLoadServer = GIsEditor || (OwnerNetMode != NM_Client);

    if (bLoadClient)

    {

        BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateClient);

    }

    if (bLoadServer)

    {

        BundlesToLoad.Add(UGameFeaturesSubsystemSettings::LoadStateServer);

    }

  

    TSharedPtr<FStreamableHandle> BundleLoadHandle = nullptr;

    if (BundleAssetList.Num() > 0)

    {

        BundleLoadHandle = AssetManager.ChangeBundleStateForPrimaryAssets(BundleAssetList.Array(), BundlesToLoad, {}, false, FStreamableDelegate(), FStreamableManager::AsyncLoadHighPriority);

    }

  

    TSharedPtr<FStreamableHandle> RawLoadHandle = nullptr;

    if (RawAssetList.Num() > 0)

    {

        RawLoadHandle = AssetManager.LoadAssetList(RawAssetList.Array(), FStreamableDelegate(), FStreamableManager::AsyncLoadHighPriority, TEXT("StartExperienceLoad()"));

    }

  

    // If both async loads are running, combine them

    TSharedPtr<FStreamableHandle> Handle = nullptr;

    if (BundleLoadHandle.IsValid() && RawLoadHandle.IsValid())

    {

        Handle = AssetManager.GetStreamableManager().CreateCombinedHandle({ BundleLoadHandle, RawLoadHandle });

    }

    else

    {

        Handle = BundleLoadHandle.IsValid() ? BundleLoadHandle : RawLoadHandle;

    }

  

    FStreamableDelegate OnAssetsLoadedDelegate = FStreamableDelegate::CreateUObject(this, &ThisClass::OnExperienceLoadComplete);

    if (!Handle.IsValid() || Handle->HasLoadCompleted())

    {

        // Assets were already loaded, call the delegate now

        FStreamableHandle::ExecuteDelegate(OnAssetsLoadedDelegate);

    }

    else

    {

        Handle->BindCompleteDelegate(OnAssetsLoadedDelegate);

  

        Handle->BindCancelDelegate(FStreamableDelegate::CreateLambda([OnAssetsLoadedDelegate]()

            {

                OnAssetsLoadedDelegate.ExecuteIfBound();

            }));

    }

```

AssetManager에 Bundle을 어쩌구 저쩌구 해서 로드가 완료 되면  OnExperienceLoadComplete 함수를 실행함

``` 

void ULyraExperienceManagerComponent::OnExperienceLoadComplete()

{

    check(LoadState == ELyraExperienceLoadState::Loading);

    check(CurrentExperience != nullptr);

  

    UE_LOG(LogLyraExperience, Log, TEXT("EXPERIENCE: OnExperienceLoadComplete(CurrentExperience = %s, %s)"),

        *CurrentExperience->GetPrimaryAssetId().ToString(),

        *GetClientServerContextString(this));

  

    // find the URLs for our GameFeaturePlugins - filtering out dupes and ones that don't have a valid mapping

    GameFeaturePluginURLs.Reset();

  

    auto CollectGameFeaturePluginURLs = [This=this](const UPrimaryDataAsset* Context, const TArray<FString>& FeaturePluginList)

    {

        for (const FString& PluginName : FeaturePluginList)

        {

            FString PluginURL;

            if (UGameFeaturesSubsystem::Get().GetPluginURLByName(PluginName, /*out*/ PluginURL))

            {

                This->GameFeaturePluginURLs.AddUnique(PluginURL);

            }

            else

            {

                ensureMsgf(false, TEXT("OnExperienceLoadComplete failed to find plugin URL from PluginName %s for experience %s - fix data, ignoring for this run"), *PluginName, *Context->GetPrimaryAssetId().ToString());

            }

        }

  

        //      // Add in our extra plugin

        //      if (!CurrentPlaylistData->GameFeaturePluginToActivateUntilDownloadedContentIsPresent.IsEmpty())

        //      {

        //          FString PluginURL;

        //          if (UGameFeaturesSubsystem::Get().GetPluginURLByName(CurrentPlaylistData->GameFeaturePluginToActivateUntilDownloadedContentIsPresent, PluginURL))

        //          {

        //              GameFeaturePluginURLs.AddUnique(PluginURL);

        //          }

        //      }

    };

  

    CollectGameFeaturePluginURLs(CurrentExperience, CurrentExperience->GameFeaturesToEnable);

    for (const TObjectPtr<ULyraExperienceActionSet>& ActionSet : CurrentExperience->ActionSets)

    {

        if (ActionSet != nullptr)

        {

            CollectGameFeaturePluginURLs(ActionSet, ActionSet->GameFeaturesToEnable);

        }

    }

  

    // Load and activate the features  

    NumGameFeaturePluginsLoading = GameFeaturePluginURLs.Num();

    if (NumGameFeaturePluginsLoading > 0)

    {

        LoadState = ELyraExperienceLoadState::LoadingGameFeatures;

        for (const FString& PluginURL : GameFeaturePluginURLs)

        {

            ULyraExperienceManager::NotifyOfPluginActivation(PluginURL);

            UGameFeaturesSubsystem::Get().LoadAndActivateGameFeaturePlugin(PluginURL, FGameFeaturePluginLoadComplete::CreateUObject(this, &ThisClass::OnGameFeaturePluginLoadComplete));

        }

    }

    else

    {

        OnExperienceFullLoadCompleted();

    }

}

```

GameFeaturePluginURLs에 CurrentExperience 과 ActionSet들의 GameFeaturesToEnable를 넣어주고  UGameFeaturesSubsystem를 통해 GameFeature들을 로드 후 실행한다. 실행이 완료된 후 에 OnGameFeaturePluginLoadComplete를 실행하게 한다. 

```

void ULyraExperienceManagerComponent::OnExperienceFullLoadCompleted()

{

    check(LoadState != ELyraExperienceLoadState::Loaded);

  

    // Insert a random delay for testing (if configured)

    if (LoadState != ELyraExperienceLoadState::LoadingChaosTestingDelay)

    {

        const float DelaySecs = LyraConsoleVariables::GetExperienceLoadDelayDuration();

        if (DelaySecs > 0.0f)

        {

            FTimerHandle DummyHandle;

  

            LoadState = ELyraExperienceLoadState::LoadingChaosTestingDelay;

            GetWorld()->GetTimerManager().SetTimer(DummyHandle, this, &ThisClass::OnExperienceFullLoadCompleted, DelaySecs, /*bLooping=*/ false);

  

            return;

        }

    }

  

    LoadState = ELyraExperienceLoadState::ExecutingActions;

  

    // Execute the actions

    FGameFeatureActivatingContext Context;

  

    // Only apply to our specific world context if set

    const FWorldContext* ExistingWorldContext = GEngine->GetWorldContextFromWorld(GetWorld());

    if (ExistingWorldContext)

    {

        Context.SetRequiredWorldContextHandle(ExistingWorldContext->ContextHandle);

    }

  

    auto ActivateListOfActions = [&Context](const TArray<UGameFeatureAction*>& ActionList)

    {

        for (UGameFeatureAction* Action : ActionList)

        {

            if (Action != nullptr)

            {

                //@TODO: The fact that these don't take a world are potentially problematic in client-server PIE

                // The current behavior matches systems like gameplay tags where loading and registering apply to the entire process,

                // but actually applying the results to actors is restricted to a specific world

                Action->OnGameFeatureRegistering();

                Action->OnGameFeatureLoading();

                Action->OnGameFeatureActivating(Context);

            }

        }

    };

  

    ActivateListOfActions(CurrentExperience->Actions);

    for (const TObjectPtr<ULyraExperienceActionSet>& ActionSet : CurrentExperience->ActionSets)

    {

        if (ActionSet != nullptr)

        {

            ActivateListOfActions(ActionSet->Actions);

        }

    }

  

    LoadState = ELyraExperienceLoadState::Loaded;

  

    OnExperienceLoaded_HighPriority.Broadcast(CurrentExperience);

    OnExperienceLoaded_HighPriority.Clear();

  

    OnExperienceLoaded.Broadcast(CurrentExperience);

    OnExperienceLoaded.Clear();

  

    OnExperienceLoaded_LowPriority.Broadcast(CurrentExperience);

    OnExperienceLoaded_LowPriority.Clear();

  

    // Apply any necessary scalability settings

#if !UE_SERVER

    ULyraSettingsLocal::Get()->OnExperienceLoaded();

#endif

}
```
CurrentExperience과 ActionSet 들의 Action들의 OnGameFeatureRegistering, OnGameFeatureLoading, OnGameFeatureActivating 를 실행해준다. 


*GameFeatures*


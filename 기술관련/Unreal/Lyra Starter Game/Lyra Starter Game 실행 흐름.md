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

GameFeaturePluginURLs에 CurrentExperience 과 ActionSet들의 GameFeaturesToEnable를 넣어주고  UGameFeaturesSubsystem를 통해 GameFeature들을 로드 후 실행한다(여기서 GameFeatures SubSystem 으로 넘어감) . 실행이 완료된 후 에 OnGameFeaturePluginLoadComplete를 실행하게 한다. 

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

```
void UGameFeaturesSubsystem::ChangeGameFeatureTargetState(const FString& PluginURL, EGameFeatureTargetState TargetState, const FGameFeaturePluginChangeStateComplete& CompleteDelegate)

{

    EGameFeaturePluginState TargetPluginState = EGameFeaturePluginState::MAX;

  

    switch (TargetState)

    {

    case EGameFeatureTargetState::Installed:    TargetPluginState = EGameFeaturePluginState::Installed;     break;

    case EGameFeatureTargetState::Registered:   TargetPluginState = EGameFeaturePluginState::Registered;    break;

    case EGameFeatureTargetState::Loaded:       TargetPluginState = EGameFeaturePluginState::Loaded;        break;

    case EGameFeatureTargetState::Active:       TargetPluginState = EGameFeaturePluginState::Active;        break;

    }

  

    // Make sure we have coverage on all values of EGameFeatureTargetState

    static_assert(std::underlying_type<EGameFeatureTargetState>::type(EGameFeatureTargetState::Count) == 4, "");

    check(TargetPluginState != EGameFeaturePluginState::MAX);

  

    const bool bIsPluginAllowed = GameSpecificPolicies->IsPluginAllowed(PluginURL);

  

    UGameFeaturePluginStateMachine* StateMachine = nullptr;

    if (!bIsPluginAllowed)

    {

        StateMachine = FindGameFeaturePluginStateMachine(PluginURL);

        if (!StateMachine)

        {

            UE_LOG(LogGameFeatures, Log, TEXT("Cannot create GFP State Machine: Plugin not allowed %s"), *PluginURL);

  

            CompleteDelegate.ExecuteIfBound(UE::GameFeatures::FResult(MakeError(UE::GameFeatures::SubsystemErrorNamespace + UE::GameFeatures::CommonErrorCodes::PluginNotAllowed)));

            return;

        }

    }

    else

    {

        StateMachine = FindOrCreateGameFeaturePluginStateMachine(PluginURL);

    }

    check(StateMachine);

  

    if (!bIsPluginAllowed)

    {

        if (TargetPluginState > StateMachine->GetCurrentState() || TargetPluginState > StateMachine->GetDestination())

        {

            UE_LOG(LogGameFeatures, Log, TEXT("Cannot change game feature target state: Plugin not allowed %s"), *PluginURL);

  

            CompleteDelegate.ExecuteIfBound(UE::GameFeatures::FResult(MakeError(UE::GameFeatures::SubsystemErrorNamespace + UE::GameFeatures::CommonErrorCodes::PluginNotAllowed)));

            return;

        }

    }

  

    if (TargetState == EGameFeatureTargetState::Active &&

        !StateMachine->IsRunning() &&

        StateMachine->GetCurrentState() == TargetPluginState)

    {

        // TODO: Resolve the activated case here, this is needed because in a PIE environment the plugins

        // are not sandboxed, and we need to do simulate a successful activate call in order run GFP systems

        // on whichever Role runs second between client and server.

  

        // Refire the observer for Activated and do nothing else.

        CallbackObservers(EObserverCallback::Activating, PluginURL, &StateMachine->GetPluginName(), StateMachine->GetGameFeatureDataForActivePlugin());

    }

    if (ShouldUpdatePluginURLData(PluginURL))

    {

        UpdateGameFeaturePluginURL(PluginURL, FGameFeaturePluginUpdateURLComplete());

    }

  

    ChangeGameFeatureDestination(StateMachine, FGameFeaturePluginStateRange(TargetPluginState), CompleteDelegate);

}

```
LoadAndActivateGameFeaturePlugin을 하면 UGameFeaturesSubsystem 에서 GameFeature에서 TargetState를 변경해줌 변경 해주고 ChangeGameFeatureDestination를 실행 

```
void UGameFeaturesSubsystem::ChangeGameFeatureDestination(UGameFeaturePluginStateMachine* Machine, const FGameFeaturePluginStateRange& StateRange, FGameFeaturePluginChangeStateComplete CompleteDelegate)

{

    const bool bSetDestination = Machine->SetDestination(StateRange,

        FGameFeatureStateTransitionComplete::CreateUObject(this, &ThisClass::ChangeGameFeatureTargetStateComplete, CompleteDelegate));

  

    if (bSetDestination)

    {

        UE_LOG(LogGameFeatures, Verbose, TEXT("ChangeGameFeatureDestination: Set Game Feature %s Destination State to [%s, %s]"), *Machine->GetGameFeatureName(), *UE::GameFeatures::ToString(StateRange.MinState), *UE::GameFeatures::ToString(StateRange.MaxState));

    }

    else

    {

        FGameFeaturePluginStateRange CurrDesitination = Machine->GetDestination();

        UE_LOG(LogGameFeatures, Display, TEXT("ChangeGameFeatureDestination: Attempting to cancel transition for Game Feature %s. Desired [%s, %s]. Current [%s, %s]"),

            *Machine->GetGameFeatureName(),

            *UE::GameFeatures::ToString(StateRange.MinState), *UE::GameFeatures::ToString(StateRange.MaxState),

            *UE::GameFeatures::ToString(CurrDesitination.MinState), *UE::GameFeatures::ToString(CurrDesitination.MaxState));

  

        // Try canceling any current transition, then retry

        auto OnCanceled = [this, StateRange, CompleteDelegate](UGameFeaturePluginStateMachine* Machine) mutable

        {

            // Special case for terminal state since it cannot be exited, we need to make a new machine

            if (Machine->GetCurrentState() == EGameFeaturePluginState::Terminal)

            {

                UGameFeaturePluginStateMachine* NewMachine = FindOrCreateGameFeaturePluginStateMachine(Machine->GetPluginURL());

                checkf(NewMachine != Machine, TEXT("Game Feature Plugin %s should have already been removed from subsystem!"), *Machine->GetPluginURL());

                Machine = NewMachine;

            }

  

            // Now that the transition has been canceled, retry reaching the desired destination

            const bool bSetDestination = Machine->SetDestination(StateRange,

                FGameFeatureStateTransitionComplete::CreateUObject(this, &ThisClass::ChangeGameFeatureTargetStateComplete, CompleteDelegate));

  

            if (!ensure(bSetDestination))

            {

                UE_LOG(LogGameFeatures, Warning, TEXT("ChangeGameFeatureDestination: Failed to set Game Feature %s Destination State to [%s, %s]"), *Machine->GetGameFeatureName(), *UE::GameFeatures::ToString(StateRange.MinState), *UE::GameFeatures::ToString(StateRange.MaxState));

  

                CompleteDelegate.ExecuteIfBound(UE::GameFeatures::FResult(MakeError(UE::GameFeatures::SubsystemErrorNamespace + UE::GameFeatures::CommonErrorCodes::UnreachableState)));

            }

            else

            {

                UE_LOG(LogGameFeatures, Display, TEXT("ChangeGameFeatureDestination: OnCanceled, set Game Feature %s Destination State to [%s, %s]"), *Machine->GetGameFeatureName(), *UE::GameFeatures::ToString(StateRange.MinState), *UE::GameFeatures::ToString(StateRange.MaxState));

            }

        };

  

        const bool bCancelPending = Machine->TryCancel(FGameFeatureStateTransitionCanceled::CreateWeakLambda(this, MoveTemp(OnCanceled)));

        if (!ensure(bCancelPending))

        {

            UE_LOG(LogGameFeatures, Warning, TEXT("ChangeGameFeatureDestination: Failed to cancel Game Feature %s"), *Machine->GetGameFeatureName());

  

            CompleteDelegate.ExecuteIfBound(UE::GameFeatures::FResult(MakeError(UE::GameFeatures::SubsystemErrorNamespace + UE::GameFeatures::CommonErrorCodes::UnreachableState + UE::GameFeatures::CommonErrorCodes::CancelAddonCode)));

        }

    }

}

```
StateMachine의 SetDestination를 실행 

```
bool UGameFeaturePluginStateMachine::SetDestination(FGameFeaturePluginStateRange InDestination, FGameFeatureStateTransitionComplete OnFeatureStateTransitionComplete, FDelegateHandle* OutCallbackHandle /*= nullptr*/)

{

    check(IsValidDestinationState(InDestination.MinState));

    check(IsValidDestinationState(InDestination.MaxState));

  

    if (!InDestination.IsValid())

    {

        // Invalid range

        return false;

    }

  

    if (CurrentStateInfo.State == EGameFeaturePluginState::Terminal && !InDestination.Contains(EGameFeaturePluginState::Terminal))

    {

        // Can't tranistion away from terminal state

        return false;

    }

  

    if (!IsRunning())

    {

        // Not running so any new range is acceptable

  

        if (OutCallbackHandle)

        {

            OutCallbackHandle->Reset();

        }

  

        FDestinationGameFeaturePluginState* CurrState = AllStates[CurrentStateInfo.State]->AsDestinationState();

  

        if (InDestination.Contains(CurrentStateInfo.State))

        {

            OnFeatureStateTransitionComplete.ExecuteIfBound(this, MakeValue());

            return true;

        }

        if (CurrentStateInfo.State < InDestination)

        {

            FDestinationGameFeaturePluginState* MinDestState = AllStates[InDestination.MinState]->AsDestinationState();

            FDelegateHandle CallbackHandle = MinDestState->OnDestinationStateReached.Add(MoveTemp(OnFeatureStateTransitionComplete));

            if (OutCallbackHandle)

            {

                *OutCallbackHandle = CallbackHandle;

            }

        }

        else if (CurrentStateInfo.State > InDestination)

        {

            FDestinationGameFeaturePluginState* MaxDestState = AllStates[InDestination.MaxState]->AsDestinationState();

            FDelegateHandle CallbackHandle = MaxDestState->OnDestinationStateReached.Add(MoveTemp(OnFeatureStateTransitionComplete));

            if (OutCallbackHandle)

            {

                *OutCallbackHandle = CallbackHandle;

            }

        }

  

        StateProperties.Destination = InDestination;

        UpdateStateMachine();

  

        return true;

    }

```
---
layout: default
title: Activate Ability
parent: Lyra Input
grand_parent: Lyra Starter Game
nav_order: 2
---

# Activate Ability from Input
라이라의 인풋시스템이 어떻게 Ability를 발동시키는지 살펴보자

LyraHeroComponent에서 바인딩 해준 함수는 Input_AbilityInputTagPressed와 Input_AbilityInputTagReleased이다.    
    
```cpp
/*
* Source/LyraGame/Character/LyraHeroComponent.cpp
*/
void ULyraHeroComponent::InitializePlayerInput(UInputComponent* PlayerInputComponent)
{

...

    if (const ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
    {
        if (const ULyraPawnData* PawnData = PawnExtComp->GetPawnData<ULyraPawnData>())
        {
            if (const ULyraInputConfig* InputConfig = PawnData->InputConfig)
            {
                
                ...

                // Ability Input Action 바인딩
                LyraIC->BindAbilityActions(InputConfig, this, &ThisClass::Input_AbilityInputTagPressed, &ThisClass::Input_AbilityInputTagReleased, /*out*/ BindHandles);
                
                ...

            }
        }
    }

...

}
```    
     
Input_AbilityInputTagReleased의 동작은 아래와 같다.
```cpp
/*
* Source/LyraGame/Character/LyraHeroComponent.cpp
*/
void ULyraHeroComponent::Input_AbilityInputTagPressed(FGameplayTag InputTag)
{
    if (const APawn* Pawn = GetPawn<APawn>())
    {
        if (const ULyraPawnExtensionComponent* PawnExtComp = ULyraPawnExtensionComponent::FindPawnExtensionComponent(Pawn))
        {
            if (ULyraAbilitySystemComponent* LyraASC = PawnExtComp->GetLyraAbilitySystemComponent())
            {
                LyraASC->AbilityInputTagPressed(InputTag);
            }
        }	
    }
}
```    
눌린 InputTag를 LyraASC의 AbilityInputTagPressed로 전달해 준다.    
     
     
ULyraAbilitySystemComponent의 AbilityInputTagPressed의 동작은 아래와 같다.    
    
```cpp
/*
* Source/LyraGame/AbilitySystem/LyraAbilitySystemComponent.cpp
*/
void ULyraAbilitySystemComponent::AbilityInputTagPressed(const FGameplayTag& InputTag)
{
    if (InputTag.IsValid())
    {
        for (const FGameplayAbilitySpec& AbilitySpec : ActivatableAbilities.Items)
        {
            if (AbilitySpec.Ability && (AbilitySpec.DynamicAbilityTags.HasTagExact(InputTag)))
            {
                InputPressedSpecHandles.AddUnique(AbilitySpec.Handle);
                InputHeldSpecHandles.AddUnique(AbilitySpec.Handle);
            }
        }
    }
}
```
즉 Pressed된 InputTag를 TArray\<FGameplayTag>타입의 멤버 변수인 InputPressedSpecHandles와 InputHeldSpecHandles에 추가해주며 해당 프레임에 눌린 InputTag를 캐싱해놓는다.
    
언리얼에서는 해당 시점의 모든 인풋처리가 끝나면 PlayerController에서 PostProcessInput함수가 불리게 되는데
Lyra에서 사용하는 LyraPlayerController는 이 함수를 ovrride해서 캐싱해놓은 InputTag를 바탕으로 Ability발동을 처리한다.

```cpp
/*
* Source/LyraGame/Player/LyraPlayerController.cpp
*/
void ALyraPlayerController::PostProcessInput(const float DeltaTime, const bool bGamePaused)
{
    if (ULyraAbilitySystemComponent* LyraASC = GetLyraAbilitySystemComponent())
    {
        LyraASC->ProcessAbilityInput(DeltaTime, bGamePaused);
    }

    Super::PostProcessInput(DeltaTime, bGamePaused);
}
```    
    
ProcessAbilityInput에서는 Ability의 Activation Policy에 따라 Ability를 발동 혹은 AbilityTask(Press 혹은 Release관련)를 시작한다.

```cpp
/*
* Source/LyraGame/Player/LyraPlayerController.cpp
*/
void ULyraAbilitySystemComponent::ProcessAbilityInput(float DeltaTime, bool bGamePaused)
{
    if (HasMatchingGameplayTag(TAG_Gameplay_AbilityInputBlocked))
    {
        ClearAbilityInput();
        return;
    }

    static TArray<FGameplayAbilitySpecHandle> AbilitiesToActivate;
    AbilitiesToActivate.Reset();


    // AbilityTask(Press 혹은 Release관련)를 시작하거나 어빌리티 인스턴스의 Handle을 AbilitiesToActivate에 캐싱한다.
    ... 


    //
    // Try to activate all the abilities that are from presses and holds.
    // We do it all at once so that held inputs don't activate the ability
    // and then also send a input event to the ability because of the press.
    //
    for (const FGameplayAbilitySpecHandle& AbilitySpecHandle : AbilitiesToActivate)
    {
        //캐싱한 어빌리티를 발동시킨다.
        TryActivateAbility(AbilitySpecHandle);
    }

    ...
    
    //
    // Clear the cached ability handles.
    //
    InputPressedSpecHandles.Reset();
    InputReleasedSpecHandles.Reset();
}
```
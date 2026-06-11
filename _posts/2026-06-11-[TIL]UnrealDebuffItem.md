---
title: "[TIL] Unreal C++ 디버프 아이템·가시 함정·AnimInstance·HUD 연동"
author: siha
date: 2026-06-11 10:20:00 +0800
categories: [TIL]
tags: [til,unreal,cpp]
render_with_liquid: false
---

## 오늘 한 것
> 슬로우·조작 반전·가시 아이템을 추가하고, `AMyPlayer` 디버프 처리와 AnimBP `MoveBlend` 연동, HUD 디버프 UI까지 마무리함. {: .prompt-success}


| 구현 내용               | 요약                                                                               |
| :---------------------- | :--------------------------------------------------------------------------------- |
| **디버프 아이템**       | `SetSlow` / `SetReverseControll` — 타이머로 해제, `Move()`에서 속도·입력 반영      |
| **가시 (`ASpikeItem`)** | 닿으면 10 데미지, 파괴 없음, 나갔다 들어오면 다시 활성화                           |
| **애니·HUD**            | `MyPlayerAnimInstance`로 `Speed` 전달, `SlowRing` / `ReverseRing`에 남은 시간 표시 |

---

## [GIT](https://github.com/siha37/SP_CH3_5Assign) 주소

## 1. `AMyPlayer` 디버프 처리

슬로우 아이템이 `SetSlow(percent, duration)`을 호출하면 이동 속도를 줄이고, `FTimerHandle`로 duration 뒤 `ResetSlow`가 돌아가게 했다. 조작 반전도 같은 방식으로 `bIsReverseControl`과 타이머를 쓴다.

HUD는 디버프가 걸릴 때 BP 함수 `SetSlowUIOnOff` / `SetReverseUIOnOff`를 호출해서 UI를 켜고, 풀릴 때 끈다. 링에 표시할 남은 시간 비율을 위해 `SlowDurationTotal`, `ReverseDurationTotal`도 같이 저장해 둠.

```cpp
void AMyPlayer::SetSlow(float percent, float duration)
{
	SlowMovePercent = FMath::Clamp(percent, 0.f, 100.f);
	SlowDurationTotal = duration;

	GetWorld()->GetTimerManager().ClearTimer(SlowTimerHandle);
	GetWorld()->GetTimerManager().SetTimer(
		SlowTimerHandle, this, &AMyPlayer::ResetSlow, duration, false);

	CallHUDSlowUI(true);
}

void AMyPlayer::Move(const FInputActionValue& value)
{
	// ... 속도 계산 후
	if (SlowMovePercent != 0)
		CurrentSpeed -= CurrentSpeed * SlowMovePercent / 100;

	GroundSpeed = CurrentSpeed * MoveInput.Size();

	if (bIsReverseControl)
		MoveInput *= -1.f;
}
```

---

## 2. 아이템 3종

**SlowingItem / ReverseControllItem** — `HealingItem`이랑 같은 패턴. 플레이어에 효과 주고 `DestroyItem()`.

**SpikeItem** — 요구사항이 달라서 구조를 나눴다. 파괴하지 않고, 오버랩 중에는 한 번만 데미지, `OnItemEndOverlap`에서 `bCanActivate`를 다시 켜서 재진입 시 다시 맞을 수 있게 함. `BaseItem`의 `OnItemOverlap` 흐름은 그대로 쓰고 `ActivateItem`만 오버라이드했다.

```cpp
void ASpikeItem::ActivateItem(AActor* OverlapActor)
{
	if (!OverlapActor || !bCanActivate) return;

	bCanActivate = false;
	UGameplayStatics::ApplyDamage(OverlapActor, SpikeDamage, nullptr, this, UDamageType::StaticClass());
}

void ASpikeItem::OnItemEndOverlap(...)
{
	if (OtherActor && OtherActor->ActorHasTag("Player"))
		bCanActivate = true;
}
```

---

## 3. AnimInstance — `MoveBlend`용 Speed

`MyPlayer`에서 `GroundSpeed`를 계산하고, `UMyPlayerAnimInstance`의 `NativeUpdateAnimation`에서 `Speed`로 넘긴다. Animation BP의 부모 클래스를 `MyPlayerAnimInstance`로 바꾸고 BlendSpace의 속도를 연결한다.

```
MyPlayer::GroundSpeed  →  AnimInstance::Speed  →  AnimBP MoveBlend
```

```cpp
void UMyPlayerAnimInstance::NativeUpdateAnimation(float DeltaSeconds)
{
	if (const AMyPlayer* Player = Cast<AMyPlayer>(TryGetPawnOwner()))
	{
		Speed = Player->GetGroundSpeed();
		bIsFalling = Player->GetIsFalling();
		bIsJumping = Player->GetIsJumping();
	}
}
```

## 배운 점

- 디버프는 로직·타이머·UI를 같이 설계하는 게 편했다. 효과만 C++에 두고 UI는 BP에 맡기되, 켜고 끄는 타이밍은 `SetSlow` / `ResetSlow`에 묶어 둠.
- 아이템도 전부 `ActivateItem` 오버라이드로 처리할 수 있지만, 가시처럼 **안 없어지는 오브젝트**는 `EndOverlap`까지 같이 봐야 한다.
- AnimBP에 값 넣는 걸 BP를 통해서 넣어줄 수도 있지만 C++로 작업해보고 싶어 시도해 본 결과
`NativeUpdateAnimation` 라는 것을 알았다.

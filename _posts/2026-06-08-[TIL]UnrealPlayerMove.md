---
title: "[TIL] Unreal C++ Custom Pawn 지상 이동체 및 3D 비행체 구현"
author: siha
date: 2026-06-08 12:20:00 +0800
categories: [TIL]
tags: [til,unreal,cpp]
render_with_liquid: false
---

## 오늘 한 것
> `ACharacter` 무브먼트 컴포넌트의 도움 없이, 순수 `APawn`을 상속받은 두 개의 서로 다른 핵심 이동 클래스(`AMyPlayer`, `AAirCraft`)를 성공적으로 완성하고 최종 디버깅을 마침. {: .prompt-success}


| 최종 구현 내용            | 핵심 기술 요소                                                                       |
| :------------------------ | :----------------------------------------------------------------------------------- |
| **1. AMyPlayer (지상형)** | 자체 중력 가속도, LineTrace 지면 검출, 시점 연동 수평 이동 및 메시 회전 인터폴레이션 |
| **2. AAirCraft (공중형)** | 6자유도(6DoF) 구현, 3D 로컬 벡터 이동, 짐벌락 방지를 위한 3축 완벽 로컬 회전 제어    |

---

## [GIT](https://github.com/siha37/SP_CH3_4Assign) 주소

## 1. 지상형 캐릭터 (`AMyPlayer`)

### 주요 트러블슈팅 및 버그 픽스

1. **카메라 롤(Roll) 오염**: `AddLocalRotation`을 매 프레임 누적할 때 쿼터니언 오차로 화면이 대각선으로 기울던 현상을, 최종 타겟 회전값을 계산해 한 번에 덮어씌우는 **`SetRelativeRotation`** 구조로 변경하여 부드러운 시점을 확보함.

### 핵심 소스 코드 정리
```cpp
void AMyPlayer::Move(const FInputActionValue& value)
{
	if(!Controller) return;
	FVector2D MoveInput = value.Get<FVector2D>();
	if (MoveInput.IsNearlyZero()) return;

	float CurrentSpeed = bIsFalling ? (MoveSpeed * AirControlModifier) : MoveSpeed;
	
	FRotator YawRot(0, Spring->GetRelativeRotation().Yaw, 0);
	FVector ForwardDirection = FRotationMatrix(YawRot).GetUnitAxis(EAxis::X) * MoveInput.Y;
	FVector RightDirection   = FRotationMatrix(YawRot).GetUnitAxis(EAxis::Y) * MoveInput.X;
	FVector MoveDirection = (ForwardDirection + RightDirection).GetSafeNormal();

	// 메시 회전 보정 및 10.0f 속도로 부드럽게 스무딩
	FRotator SmoothRotation = FMath::RInterpTo(Mesh->GetRelativeRotation() + FRotator(0,90,0), MoveDirection.Rotation(), GetWorld()->GetDeltaSeconds(), 10.0f);
	Mesh->SetRelativeRotation(SmoothRotation + FRotator(0,-90,0));

	AddActorWorldOffset(MoveDirection * CurrentSpeed * GetWorld()->GetDeltaSeconds(), true);
}
```

---

## 2. 공중 자유 비행체 구현 (`AAirCraft`)

### 주요 트러블슈팅 및 버그 픽스
1. **월드 회전 혼용으로 인한 비행축 뒤틀림**: 마우스 좌우 입력(`Yaw`) 처리 시 지면 기준의 `AddActorWorldRotation`을 사용하는 바람에 기체가 사선이나 배면 비행 상태일 때 시점이 대각선으로 요동치는 하이퍼 짐벌락 발생.
2. **해결책**: 3D 공간을 자유롭게 비행하는 기체는 지구 중심의 월드 좌표계에 종속되면 안 되므로, **모든 3축 회전(Pitch, Yaw, Roll)을 완벽하게 동체 기준의 `AddActorLocalRotation`으로 일괄 통일**하여 비행기가 거꾸로 뒤집혀도 기수 방향 기준 조작이 정상 작동하도록 제어함.

### 핵심 소스 코드 정리
```cpp
void AAirCraft::Look(const FInputActionValue& value)
{
	FVector2D LookDirection = value.Get<FVector2D>();

	float DeltaPitch = LookDirection.Y * PitchSpeed * GetWorld()->GetDeltaSeconds();
	float DeltaYaw = LookDirection.X * YawSpeed * GetWorld()->GetDeltaSeconds();

	// 비행체는 기체 기준의 완벽한 Local 회전만을 결합해야 대각선 꼬임이 일어나지 않는다.
	AddActorLocalRotation(FRotator(DeltaPitch, DeltaYaw, 0.0f));
}

void AAirCraft::Roll(const FInputActionValue& value)
{
	float RollInput = value.Get<float>();
	float DeltaRoll = RollInput * RollSpeed * GetWorld()->GetDeltaSeconds();

	AddActorLocalRotation(FRotator(0.0f, 0.0f, DeltaRoll));
}
```

---

## 최종 성과 및 배운 점

### 1. 로컬(Local) 좌표계와 월드(World) 좌표계의 명확한 경계 인식
* 이동 거리가 절대 방향(컨트롤러/카메라 뷰 방향) 기준일 때는 **WorldOffset**, 기체가 바라보는 방향 기준일 때는 **LocalOffset**을 적용해야 물리적 오동작이 없음을 깨달았다.

### 2. 3D 공간 회전 연산의 수학적 오염 차단
* 상대적 회전 누적(`Add`) 연산이 불러오는 내부 쿼터니언 축 오염(`Roll` 전이 현상)을 해결하기 위해 정제된 회전 변수를 활용하는 구조 설계 능력을 배양했다.

### 3. 무브먼트 컴포넌트 독립 프로세스 달성
* 엔진이 제공하는 블랙박스형 기능(`CharacterMovement`)에 의존하지 않고, `Tick` 내부의 중력 등가속도 연산($v = v_0 + at$)과 지면 레이캐스트 검출을 직접 바인딩하며 언리얼 내부 프레임 단위 물리 업데이트 구조를 만들어봤다.

---
title: "[TIL] Unreal C++ 아이템 시스템 (Interface & 상속 구조) 및 Collision 응답 트러블슈팅"
author: siha
date: 2026-06-09 12:20:00 +0800
categories: [TIL]
tags: [til,unreal,cpp]
render_with_liquid: false
---

## 오늘 한 것
> C++ `Interface`(`IItemInterface`)와 상속 계층을 활용해 확장 가능한 아이템 시스템을 설계하고, 오버랩 기반 획득/폭발 로직을 구현함. 커스텀 `APawn` 이동에서 사소한 문제로 발생한 이동 불가 버그의 원인을 찾음. {: .prompt-success}

| 최종 구현 내용                   | 핵심 기술 요소                                                               |
| :------------------------------- | :--------------------------------------------------------------------------- |
| **1. 아이템 시스템 (Interface)** | `IItemInterface` 순수 가상 계약, `ABaseItem` 공통 구현, 다단계 상속 + 다형성 |
| **2. AMineItem (지연 폭발)**     | `FTimerHandle` 기반 딜레이, 별도 `ExplosionCollision` 범위                   |
| **3. Collision 트러블슈팅**      | Block vs Overlap 응답 차이, Sweep 이동과 충돌 응답의 상관관계 규명           |

---

## [GIT](https://github.com/siha37/SP_CH3_5Assign) 주소

## 1. 아이템 시스템 설계 (Interface + 상속)

### 설계 의도
* 모든 아이템이 공통적으로 가져야 할 행위(`OnItemOverlap`, `ActivateItem`, `GetItemType`)를 **순수 가상 인터페이스**로 강제하고, 실제 공통 동작은 `ABaseItem`에 한 번만 구현해 중복을 제거함.


### 핵심 소스 코드 정리
```cpp
// IItemInterface : 모든 아이템이 지켜야 할 순수 가상 계약
class SP_CH3_5ASSIGN_API IItemInterface
{
    GENERATED_BODY()
public:
    virtual void OnItemOverlap(UPrimitiveComponent* OverlappedComp, AActor* OtherActor,
        UPrimitiveComponent* OtherComp, int32 OtherBodyIndex,
        bool bFromSweep, const FHitResult& SweepResult) = 0;
    virtual void ActivateItem(AActor* OverlapActor) = 0;
    virtual FName GetItemType() const = 0;
};

// ABaseItem : 공통 구현 (충돌 컴포넌트 구성 + 델리게이트 바인딩)
ABaseItem::ABaseItem()
{
    Scene = CreateDefaultSubobject<USceneComponent>("Scene");
    SetRootComponent(Scene);

    Collision = CreateDefaultSubobject<USphereComponent>("Collision");
    Collision->SetCollisionProfileName(TEXT("OverlapAll")); // 트리거 역할
    Collision->SetupAttachment(Scene);

    Mesh = CreateDefaultSubobject<UStaticMeshComponent>("Mesh");
    Mesh->SetCollisionProfileName(TEXT("NoCollision"));      // 시각 전용
    Mesh->SetupAttachment(Collision);

    Collision->OnComponentBeginOverlap.AddDynamic(this, &ABaseItem::OnItemOverlap);
    Collision->OnComponentEndOverlap.AddDynamic(this, &ABaseItem::OnItemEndOverlap);
}

void ABaseItem::OnItemOverlap(UPrimitiveComponent* OverlappedComp, AActor* OtherActor, /* ... */)
{
    if (OtherActor && OtherActor->ActorHasTag("Player"))
    {
        ActivateItem(OtherActor); // 자식 클래스에서 오버라이드된 동작 호출
    }
}
```

---

## 2. AMineItem - 타이머 기반 지연 폭발

1. **즉발이 아닌 지연 처리**: 지뢰는 밟는 즉시 사라지면 안 되고 일정 시간 뒤 폭발해야 하므로, `Tick` 누적 카운트 대신 **`FTimerManager::SetTimer`** 로 `ExplosionDelay` 이후 단발 콜백을 예약하는 구조로 구현함.
2. **획득 판정, 폭발 판정**: 플레이어가 닿는 트리거 `Collision`과, 폭발 시점에 광역 피해 대상을 찾는 `ExplosionCollision`(반경 `ExplosionRadius`)을 **별도 컴포넌트로 분리**하여, 닿는 범위와 터지는 범위를 독립적으로 제어함.

### 핵심 소스 코드 정리
```cpp
AMineItem::AMineItem()
{
    ExplosionRadius = 300.f;
    ExplosionDelay  = 5.f;
    ExplosionDamage = 30;
    ItemType = "Mine";

    ExplosionCollision = CreateDefaultSubobject<USphereComponent>(TEXT("ExplosionCollision"));
    ExplosionCollision->InitSphereRadius(ExplosionRadius);
    ExplosionCollision->SetCollisionProfileName(TEXT("OverlapAllDynamic"));
    ExplosionCollision->SetupAttachment(Scene);
}

void AMineItem::ActivateItem(AActor* OverlapActor)
{
    // 밟은 즉시 ExplosionDelay 후 Explode() 1회 예약
    GetWorld()->GetTimerManager().SetTimer(
        TimerHandle, this, &AMineItem::Explode, ExplosionDelay, false);
}

void AMineItem::Explode()
{
    TArray<AActor*> OverlappingActors;
    ExplosionCollision->GetOverlappingActors(OverlappingActors); // 범위 내 대상 일괄 수집

    for (AActor* Actor : OverlappingActors)
    {
        if (Actor && Actor->ActorHasTag("Player"))
        {
            // 데미지 처리 지점
        }
    }
    DestroyItem();
}
```

---

## 3. 트러블슈팅: Block vs Overlap Collision과 커스텀 무브먼트

### 증상
* `AMyPlayer`(순수 `APawn`, `CharacterMovement` 미사용)의 캡슐 Collision Preset을 **BlockAllDynamic**으로 설정하자, Move 로직은 호출되는데 실제 위치 이동이 전혀 일어나지 않음.
* 동일 코드에서 **OverlapAllDynamic**으로 바꾸면 정상적으로 이동함.
* 이후 1차 해결 후 이동은 가능해졌으나 속도가 느려짐.

### 원인 규명
1. 처음 Player 객체가 생성될 때 지면보다 살짝 아래에서 스폰이 됨.
2. 땅 속에 박혀있는 판정으로 인해 이동이 불가해짐.
3. Player 시작 위치 Z를 위로 올림.
4. 이동은 가능해졌지만 점프가 안됨.
5. 기존 방식으론 Collision이 아닌 Trace로 중력처리를 하였지만 충돌 처리를 Collision에 맞기면서 Trace의 거리가 Collision 크기 보다 작아 바닥 인식 처리 자체가 안됨.

### 해결
1. 플레이어 시작 위치를 조정
2. Trace의 `Distance` 값을 Collision보다 길게 만듬

## 최종 성과 및 배운 점

### 1. Interface를 통해 기능을 강제하는 구조
* "무엇을 해야 하는가(인터페이스)"와 "어떻게 하는가(BaseItem 구현)"를 분리하고, 다단계 상속으로 공통 로직을 재사용하면서 자식에서 `ActivateItem`만 오버라이드하는 다형성 구조를 설계해봤다.
* 몰론 해당 방식은 상속의 깊이가 깊어지면 관리가 어렵다 느낌.

### 3. 코드 로직은 문제 없지만 단순한 수치만으로 생기는 문제
* 코드 상으로는 전혀 문제가 없었음에도 오히려 단순한 오류여서 찾기 힘들었던 문제이다.
* 다음에는 테스트를 위한 기즈모를 추가하여 테스트하는 과정이 필요하다 느낌.

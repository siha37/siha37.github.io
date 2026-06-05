---
title: "[Unreal]Actor Spawn"
author: siha
date: 2026-06-05 19:44:00 +0800
categories: [Unreal]
tags: [til,unreal]
render_with_liquid: false
---

## SpawnActor 메서드
Actor를 스폰하기 위해 UWorld::SpawnActor()를 사용한다.
다양한 상황을 고려하여 많은 함수로 오버로드 되어 있다.

```
	AActor* UWorld::SpawnActor
	(
    // 스폰 시킬 Actor 클래스
		UClass*			Class,

    // 스폰할 객체 이름 - 없으면 자동 생성
		FName			InName,

    // 스폰 위치
		FVector const*	Location,

    // 스폰 시 회전값
		FRotator const*	Rotation,

    // 새 Actor를 스폰 시 AActor의 초기화 처리 - 자세히 알아볼 것 ??
		AActor*			Template,

    // true - 콜리전 테스트 ON / false - 콜리전 테스트 OFF
		bool			bNoCollisionFail,

    // ?
		bool			bRemoteOwned,

    // 스폰된 Actor를 소유하는 AActor
		AActor*			Owner,
    
    // 스폰된 Actor가 입힌 피해를 담당하는 APawn ??
		APawn*			Instigator,

    // 스폰이 조건이 충족하지 않아도 스폰 시키는 여부 설정
		bool			bNoFail,

    // 어떤 레벨에 스폰시킬지 설정
		ULevel*			OverrideLevel,

    // Construction Script 실행 여부 ??
		bool			bDeferConstruction
	)
```

## 사용법

기본적으론 아래와 같이 사용 가능
```
AMyPlayer* SpawnPlayer = GetWorld()->SpawnActor(AMyPlayer::StaticClass(),location,rotation);
```

T제너릭을 통해 오버로드 형태이다. 이외에 여러 형태가 있다.
```
	template< class T >
	T* SpawnActor
	(
		UClass* Class,
		AActor* Owner=NULL,
		APawn* Instigator=NULL,
		bool bNoCollisionFail=false
	)
	{
		return (Class != NULL) ? Cast<T>(GetWorld()->SpawnActor(Class, NAME_None, NULL, NULL, NULL, bNoCollisionFail, false, Owner, Instigator)) : NULL;
	}
```

## 느낀 점
일부 매개변수의 경우 용도를 명확히 인지하지 못해 공부가 필요하다 느낌




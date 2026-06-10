---
title: "[TIL] Unreal C++ DataTable 기반 Wave 시스템 및 아이템 수집 게임 루프 구현"
author: siha
date: 2026-06-10 19:21:00 +0900
categories: [TIL]
tags: [til,unreal,cpp]
render_with_liquid: false
---

## 오늘 한 것
> `APlayerGameState`를 중심으로 **DataTable 기반 Wave/레벨 진행 시스템**을 완성하고, 코인 수집·지뢰·회복 아이템 상호작용, HUD/메인메뉴 UI, 플레이어 체력·데미지 처리까지 전체 게임 루프를 연결함. {: .prompt-success}


| 최종 구현 내용                          | 핵심 기술 요소                                                                                                    |
| :-------------------------------------- | :---------------------------------------------------------------------------------------------------------------- |
| **1. Wave 시스템 (`APlayerGameState`)** | `LevelMapData` → `FWaveRow` → `SpawnRow` → `FItemSpawnRow` 3계층 DataTable, 웨이브별 타이머·스폰·클리어 조건 분기 |
| **2. 아이템 & 스폰 (`ASpawnValume`)**   | 가중치 랜덤 스폰, Box Volume 내 랜덤 좌표 배치, 코인/회복/지뢰 파생 클래스별 효과 처리                            |
| **3. 플레이어 & UI (`AMyPlayer`)**      | `TakeDamage`·오버헤드 HP 위젯, 사망 시 `OnGameOver`, HUD(시간·점수·레벨/웨이브) 갱신                              |

---

## [GIT](https://github.com/siha37/SP_CH3_5Assign) 주소

## 1. DataTable 기반 Wave 시스템 (`APlayerGameState`)

### 주요 트러블슈팅 및 버그 픽스

1. **단일 스폰 테이블 한계**: 기존 `ItemSpawnTable` 하나로는 레벨·웨이브별 아이템 수·제한 시간을 다르게 줄 수 없어, `LevelMapData`(레벨별 `FWaveRow` 테이블) + `SpawnRow`(웨이브별 `FItemSpawnRow` 테이블) **2단 DataTable 구조**로 분리함.
2. **웨이브 종료 타이머 중복**: 코인 전량 수집으로 웨이브를 조기 클리어할 때 `LevelTimerHandle`이 남아 다음 웨이브에서 시간이 꼬이던 문제를, `EndWave(bool bClearLevelTimer)` 플래그로 **성공 시에만 타이머를 명시적으로 Clear**하도록 수정함.
3. **레벨 전환 시 타이머 누수**: `EndPlay` / `OnGameOver`에서 `ClearAllTimersForObject(this)`를 호출해, 맵 전환·게임 오버 후에도 HUD/웨이브 타이머가 살아남는 현상을 차단함.

### 핵심 소스 코드 정리

```cpp
void APlayerGameState::StartWave()
{
	SpawnedCoinCount = 0;
	CollectedCoinCount = 0;
	ClearSpawnedItems();

	FWaveRow* Wave = GetWaveData();
	UDataTable* ItemData = GetItemData();

	for (int32 i = 0; i < Wave->ItemAmount; ++i)
	{
		if (ASpawnValume* SpawnVolume = Cast<ASpawnValume>(FoundVolumes.Last()))
		{
			if (AActor* SpawnedActor = SpawnVolume->SpawnRandomItem(ItemData))
			{
				if (SpawnedActor->IsA(ACoinItem::StaticClass()))
					++SpawnedCoinCount;
			}
		}
	}

	const float Duration = Wave->WaveDuration > 0.f ? Wave->WaveDuration : LevelDuration;
	GetWorldTimerManager().SetTimer(
		LevelTimerHandle,
		this,
		&APlayerGameState::OnLevelTimeUp,
		Duration,
		false
	);
}

void APlayerGameState::EndWave(bool bClearLevelTimer)
{
	if (bClearLevelTimer)
		GetWorldTimerManager().ClearTimer(LevelTimerHandle);

	CurrentWaveCount++;
	if (CurrentWaveCount < GetMaxWavesForCurrentLevel())
	{
		StartWave();
		return;
	}
	EndLevel();
}
```

### Level / Wave / ItemSpawn DataTable 구조

게임 데이터는 **3계층**으로 연결된다.

```
LevelMapData[레벨 인덱스]     ← FWaveRow 테이블 (레벨당 1개)
  └─ 행[웨이브 인덱스]        ← 1행 = 1웨이브
       └─ SpawnRow            ← FItemSpawnRow 테이블 (웨이브당 1개)
            └─ 아이템 클래스 + 확률
```

| 계층          | Struct          | 담당                                               |
| :------------ | :-------------- | :------------------------------------------------- |
| **Level**     | `FWaveRow`      | 웨이브별 스폰 수, 제한 시간, ItemSpawn 테이블 참조 |
| **ItemSpawn** | `FItemSpawnRow` | 스폰할 아이템 클래스와 가중치 확률                 |
| **맵 전환**   | `LevelMapNames` | 레벨 클리어 시 이동할 맵 이름 목록                 |

- `CurrentLevelIndex` → `LevelMapData`에서 레벨 선택
- `CurrentWaveCount` → 해당 Level 테이블의 행(웨이브) 선택
- `FWaveRow.SpawnRow` → 이번 웨이브의 아이템 풀 참조
- 모든 웨이브 클리어 → `LevelMapNames`로 다음 맵 이동

---

## 2. 가중치 랜덤 아이템 스폰 (`ASpawnValume`)

### 주요 트러블슈팅 및 버그 픽스

1. **스폰 위치 편향**: 고정 좌표 스폰 대신 `UBoxComponent`의 `ScaledBoxExtent`를 활용해 **볼륨 내부 균등 랜덤 좌표**를 계산, 맵 전역에 아이템이 고르게 퍼지도록 함.
2. **확률 테이블 합산**: `Spawnchance` 누적합 방식으로 가중치 랜덤을 구현해, 에디터에서 행별 확률만 조정하면 코인·지뢰·회복 비율을 웨이브마다 다르게 설정 가능.

### 핵심 소스 코드 정리

```cpp
FItemSpawnRow* ASpawnValume::GetRandomItem(UDataTable* ItemDataTable)
{
	TArray<FItemSpawnRow*> Rows;
	ItemDataTable->GetAllRows(ContextString, Rows);

	float TotalChance = 0.0f;
	for (const FItemSpawnRow* Row : Rows)
		TotalChance += Row->Spawnchance;

	const float RandValue = FMath::RandRange(0.0f, TotalChance);
	float AccumulatedChance = 0.0f;

	for (FItemSpawnRow* Row : Rows)
	{
		AccumulatedChance += Row->Spawnchance;
		if (AccumulatedChance >= RandValue)
			return Row;
	}
	return nullptr;
}
```

---

## 3. 아이템 상호작용 & 플레이어 생존 시스템

### 주요 트러블슈팅 및 버그 픽스

1. **지뢰 중복 폭발**: `Overlap`이 연속 발생할 때 타이머가 중복 등록되던 문제를 `bHasExploded` 플래그와 `EndPlay`에서의 `TimerHandle` Clear로 방지함.
2. **파티클 잔존**: `SpawnEmitterAtLocation`으로 생성된 파티클이 액터 파괴 후에도 남는 문제를, `TWeakObjectPtr` + 람다 타이머로 **지연 `DestroyComponent`** 처리함.
3. **점수 영속성**: 레벨 전환 시 점수가 초기화되지 않도록 `UMyGameInstance::TotalScore`에 누적하고, 레벨 종료 시 `CurrentLevelIndex`도 함께 저장해 **맵 간 상태를 유지**함.

### 핵심 소스 코드 정리

```cpp
void ACoinItem::ActivateItem(AActor* OverlapActor)
{
	Super::ActivateItem(OverlapActor);
	if (OverlapActor && OverlapActor->ActorHasTag("Player"))
	{
		if (APlayerGameState* state = GetWorld()->GetGameState<APlayerGameState>())
		{
			state->AddScore(PointValue);
			state->OnCoinCollected();
		}
		DestroyItem();
	}
}

void AMineItem::Explode()
{
	// VFX / SFX 재생 후
	for (AActor* actor : OverlappingActors)
	{
		if (actor && actor->ActorHasTag("Player"))
		{
			UGameplayStatics::ApplyDamage(
				actor,
				ExplosionDamage,
				nullptr,
				this,
				UDamageType::StaticClass()
			);
		}
	}
	DestroyItem();
}
```

---

## 최종 성과 및 배운 점

### 1. GameState를 중심으로 한 게임 루프 설계
* `AGameState`는 레벨 전환 후 재생성되므로, **영속 데이터(점수·레벨 인덱스)는 `UGameInstance`**, **웨이브·타이머·스폰 상태는 `APlayerGameState`**로 역할을 분리하는 것이 자연스러움을 확인함.

### 2. DataTable 2단 구조의 확장성
* `FWaveRow`(웨이브 메타: 아이템 수, 제한 시간, 스폰 테이블 참조)와 `FItemSpawnRow`(아이템 클래스, 확률)를 분리하면 **코드 수정 없이 에디터에서 난이도 곡선**을 조정할 수 있음을 체감함.

### 3. 타이머·위젯·파티클의 생명주기 관리
* `FTimerHandle`은 액터/GameState 파괴 시 반드시 Clear해야 하며, `SpawnEmitterAtLocation`으로 생성한 이펙트는 액터와 별개 생명주기를 가지므로 **`TWeakObjectPtr`로 안전하게 정리**해야 함을 배움.

### 4. 승리/패배 조건의 이중 분기
* **시간 초과**(`OnLevelTimeUp` → `EndWave(false)`)와 **코인 전량 수집**(`OnCoinCollected` → `EndWave(true)`) 두 경로가 같은 `EndWave` 파이프라인을 공유하되, 타이머 Clear 여부만 다르게 처리하는 패턴으로 웨이브 전환 로직을 단순화함.

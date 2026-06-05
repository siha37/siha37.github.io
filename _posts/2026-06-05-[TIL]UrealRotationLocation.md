---
title: "[TIL]Unreal C++ Actor 조작"
author: siha
date: 2026-06-05 19:44:00 +0800
categories: [TIL]
tags: [til,unreal]
render_with_liquid: false
---

## 오늘 한 것

>Unreal C++ 프로젝트 내에서 Pawn 클래스를 상속 받아 회전, 이동 코드를 작성
>IMC를 새로 만든 Pawn클래스에 맵핑 시도  {: .prompt-info}

| 오늘 한 것                         |
| ---------------------------------- |
| Pawn 객체 회전,이동                |
| Actor 스폰                         |
| Timer 사용해보기                   |
| Debug 코드 추가                    |
| IMC를 PlayerControll 클래스에 매핑 |

## [Pawn 객체 회전, 이동](/posts/PawnRotationLocation)

```
// AObstacle_Type0.h
UCLASS()
class SP_CH3_3ASSIGN_API AObstacle_Type0 : public APawn
{
	GENERATED_BODY()

public:
	// Sets default values for this pawn's properties
	AObstacle_Type0();


protected:
	
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly,Category="Components")
	USceneComponent* Root;
	
	UPROPERTY(VisibleAnywhere,BlueprintReadOnly,Category="Components")
	UStaticMeshComponent* StaticMesh;

	UPROPERTY(EditAnywhere,Blueprintable,Category="Rotate")
	float RotationSpeed = 20.0f;
	
	virtual void BeginPlay() override;

	

public:	
	// Called every frame
	virtual void Tick(float DeltaTime) override;

	// Called to bind functionality to input
	virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;

};
```

```
// AObstacle_Type0.cpp
// Sets default values
AObstacle_Type0::AObstacle_Type0()
{
	Root = CreateDefaultSubobject<USceneComponent>("Root");
	StaticMesh = CreateDefaultSubobject<UStaticMeshComponent>("StaticMesh");
	StaticMesh->SetupAttachment(Root);

	PrimaryActorTick.bCanEverTick = true;
}

void AObstacle_Type0::BeginPlay()
{
	Super::BeginPlay();
}

// Called every frame
void AObstacle_Type0::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

	if(!FMath::IsNearlyZero(RotationSpeed))
		AddActorLocalRotation(FRotator(0, RotationSpeed * DeltaTime, 0));

}

// Called to bind functionality to input
void AObstacle_Type0::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);

}
```

```
// AObstacle_Type1.h
class SP_CH3_3ASSIGN_API AObstacle_Type1 : public APawn
{
	GENERATED_BODY()

public:
	// Sets default values for this pawn's properties
	AObstacle_Type1();

protected:
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly,Category="Components")
	USceneComponent* Root;
	
	UPROPERTY(VisibleAnywhere,Blueprintable,Category="Components")
	UStaticMeshComponent* StaticMesh;

	UPROPERTY(EditAnywhere,BlueprintReadOnly,Category = "Move")
	FVector StartPosition;
	UPROPERTY(EditAnywhere,Blueprintable,Category = "Move")
	FVector Direction;
	UPROPERTY(EditAnywhere,Blueprintable,Category = "Move")
	float MoveSpeed;
	UPROPERTY(EditAnywhere,Blueprintable,Category = "Move")
	float MaxRange;

	FTimerHandle TimerHandle;
	float TimeRate = 0.016f;

	// Called when the game starts or when spawned
	virtual void BeginPlay() override;

public:
	bool IsRevers = false;
	
	virtual void Tick(float DeltaTime) override;
	virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;

	void MoveActor();
};
```
```
// AObstacle_Type1.cpp
// Sets default values
AObstacle_Type1::AObstacle_Type1()
{
	Root = CreateDefaultSubobject<USceneComponent>("Root");
	StaticMesh = CreateDefaultSubobject<UStaticMeshComponent>("StaticMesh");
	StaticMesh->SetupAttachment(Root);

	PrimaryActorTick.bCanEverTick = false;
}

// Called when the game starts or when spawned
void AObstacle_Type1::BeginPlay()
{
	Super::BeginPlay();
	StartPosition = GetActorLocation();
	GetWorld()->GetTimerManager().SetTimer(TimerHandle,this,&AObstacle_Type1::MoveActor,TimeRate,true);
}

// Called every frame
void AObstacle_Type1::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
}

// Called to bind functionality to input
void AObstacle_Type1::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);
	
}

void AObstacle_Type1::MoveActor()
{
	float distance = FVector::Distance(GetActorLocation(), StartPosition);
	
	// 조건 거리 충족
	if(MaxRange <= distance)
	{
		IsRevers = !IsRevers;
		StartPosition = GetActorLocation();
	}
	FVector dynamicDirection = Direction * MoveSpeed * TimeRate * (IsRevers? 1:-1);
	AddActorWorldOffset(dynamicDirection);
}

```

## [Actor Spawn](/posts/ActorSpawn)

```
// AObstacleSpawner.h
UCLASS()
class SP_CH3_3ASSIGN_API AObstacleSpawner : public AActor
{
	GENERATED_BODY()
	
public:	
	// Sets default values for this actor's properties
	AObstacleSpawner();

protected:
	// Called when the game starts or when spawned
	virtual void BeginPlay() override;

	UPROPERTY(EditAnywhere,BlueprintReadWrite, Category = "Spawning")
	//AActor의 하위 클래스만 가능하도록 TSubclassOf로 안정성을 추가
	TArray<TSubclassOf<AActor>> SpawnActors;
	
	void SpawnAllActors();

public:	
	// Called every frame
	virtual void Tick(float DeltaTime) override;

};
```
```
// AObstacleSpawner.cpp
AObstacle_Type1::AObstacle_Type1()
{
	Root = CreateDefaultSubobject<USceneComponent>("Root");
	StaticMesh = CreateDefaultSubobject<UStaticMeshComponent>("StaticMesh");
	StaticMesh->SetupAttachment(Root);

	PrimaryActorTick.bCanEverTick = false;
}

// Called when the game starts or when spawned
void AObstacle_Type1::BeginPlay()
{
	Super::BeginPlay();
	StartPosition = GetActorLocation();
	GetWorld()->GetTimerManager().SetTimer(TimerHandle,this,&AObstacle_Type1::MoveActor,TimeRate,true);
}

// Called every frame
void AObstacle_Type1::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
}

// Called to bind functionality to input
void AObstacle_Type1::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);
	
}

void AObstacle_Type1::MoveActor()
{
	float distance = FVector::Distance(GetActorLocation(), StartPosition);
	
	// 조건 거리 충족
	if(MaxRange <= distance)
	{
		IsRevers = !IsRevers;
		StartPosition = GetActorLocation();
	}
	FVector dynamicDirection = Direction * MoveSpeed * TimeRate * (IsRevers? 1:-1);
	AddActorWorldOffset(dynamicDirection);
}
```

## [Tick 함수 대신 Timer 사용하기](/posts/TimerManager)

```

	FTimerHandle TimerHandle;
	float TimeRate = 0.016f;


// Called when the game starts or when spawned
void AObstacle_Type1::BeginPlay()
{
	Super::BeginPlay();
	StartPosition = GetActorLocation();
	GetWorld()->GetTimerManager().SetTimer(TimerHandle,this,&AObstacle_Type1::MoveActor,TimeRate,true);
}

void AObstacle_Type1::MoveActor()
{
	float distance = FVector::Distance(GetActorLocation(), StartPosition);
	
	// 조건 거리 충족
	if(MaxRange <= distance)
	{
		IsRevers = !IsRevers;
		StartPosition = GetActorLocation();
	}
	FVector dynamicDirection = Direction * MoveSpeed * TimeRate * (IsRevers? 1:-1);
	AddActorWorldOffset(dynamicDirection);
}
```

## [Debug 코드 사용해보기](/posts/DebugLog)

```
// .h
DECLARE_LOG_CATEGORY_EXTERN(SPObstacle, Display, All);

// .cpp
DEFINE_LOG_CATEGORY(SPObstacle);

void AObstacle_Type3::SetOnOff()
{
	StaticMesh->SetVisibility(!StaticMesh->IsVisible());
	TimeRate = FMath::RandRange(MinTimeRate,MaxTimeRate);
	UE_LOG(LogTemp,Display,TEXT("Time Rate: %f"),TimeRate);
	GetWorld()->GetTimerManager().SetTimer(TimerHandle,this,&AObstacle_Type3::SetOnOff,TimeRate,false);

```

## [SetActive & Activate](/posts/SetActive_Activate)
```
StaticMesh->SetVisibility(!StaticMesh->IsVisible());
```

## 배운 점

### 배움
객체의 회전과 이동 
Tick이 아닌 타이머를 통한 Update
로그 출력
활성화와 비활성화의 미묘한 차이

### 의미
직접 과제를 진행하며 듣고 보는 것보다 익숙해지기 위한 시간을 보냈다.

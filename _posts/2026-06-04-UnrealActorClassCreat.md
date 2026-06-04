---
title: Unreal Actor Class 생성
author: cotes
date: 2026-06-04 19:10:00 +0800
categories: [Unreal]
tags: [til,unreal]
render_with_liquid: false
---

## C++ Class 추가

1. Unreal 에디터 상단 Tool-> New C++ Class 선택
2. Actor를 클릭하고 이름을 설정
3. ClassType을 Public 또는 Private 설정
4. Create Class 클릭

![ActorClass 추가](/assets/img/26-06-04/스크린샷%202026-06-04%20200640.png)


## C++ 클래스 생성 위치

방금 생성한 소스코드는 .h는 Public 폴더에  .cpp는 Private 폴더에 들어가며
다른 모듈에서 쉽게 접근 가능해 진다.

반대로 Private를 설정하여 생성하면 .h .cpp 둘다 Private 폴더에 들어가며 외부 노출이 안된다.

## 클래스 삭제
>Unity는 단순하게 스크립트 파일을 제거하면 바로 적용이 되지만 Unreal은 과정이 복잡하다.

1. Visual Studio에서 소스코드 파일을 제거
2. 파일 탐색기를 통해 해당 파일을 삭제
3. Visual Studio 빌드
4. 에디터 재실행

## Actor Class 파악하기

```
#pragma once

// Unreal 기본 헤더, 엔진 전역 타입,메크로,함수 등을 포함함
#include "CoreMinimal.h

// AActor 클래스 선언을 가능하게 함
#include "GameFramework/Actor.h"

// Unreal 헤더툴(UHT)이 자동 생성하는 코드를 포함
// 무조건 마지막 줄에 위치 해야함
#include "Item.generated.h"
```



```
UCLASS()
class CH3_L_SP_API AAitem : public AActor
{
	GENERATED_BODY()
	
public:	
	AAitem();

protected:
  // 객체가 월드에서 시작되고 한번 호출
	virtual void BeginPlay() override;

public:	
  // 매 프레임 호출 함수
	virtual void Tick(float DeltaTime) override;

};
```

## UCLASS()
해당 클래스를 리플렉션 시스템에 등록하는 매크로
블루프린트로 확장 가능하게 함

## CH3_L_SP_API
프로젝트명마다 달라지며 모듈 외부에서도 클래스를 사용할 수 있게 하는 메크로이다.

## GENERATED_BODY()
UCLASS()와 짝을 이루는 매크로 * 더 공부할 내용



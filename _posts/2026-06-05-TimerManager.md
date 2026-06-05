---
title: "[Unreal]TimerManager"
author: siha
date: 2026-06-05 19:44:00 +0800
categories: [Unreal]
tags: [til,unreal]
render_with_liquid: false
---

## 타이머
FTimerManager의 SetTimer 함수를 통해 설정한 딜레이 이후 함수나 델리게이트를 호출하는 타이머를 설정하고
반복되게 할 수 있다.

## TimerManager에 접근법
TimerManager는 GameInstance 와 World에 존재한다.

접근은 아래와 같다
```
//타이머 매니저 반환
GetWorld()->GetTimerManager()
//or
AActor::GetTimerManager()

```

## SetTimer

해당 함수를 통해서 타이머를 세팅한다
사용법은 아래와 같다
```
GetWorldTimerManager().SetTimer(
  // FTimerHandle 형태 변수 - 이후 컨트롤를 위해
  MemberTimerHandle,

  // 호출하고자 하는 함수의 객체
  this,

  // 호출하고자 하는 함수 주소
  &AMyActor::RepeatingFunction,

  // 딜레이 시간
  1.0f,

  // 루프 유무
  true,

  // 처음 시작 시 딜레이 - 기본값 = -1
  2.0f);


```

## PauseTimer(FTimerHandle)
타이머를 일시정지 시키는 함수
```
GetWorld()->GetTimerManager().PauseTimer(TimerHandle);
```

## UnPauseTimer(FTimerHandle)
타이머를 재개하는 함수
```
GetWorld()->GetTimerManager().UnPauseTimer(TimerHandle);
```

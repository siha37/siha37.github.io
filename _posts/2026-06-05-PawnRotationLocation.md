---
title: "[Unreal]Rotation&Location 함수"
author: siha
date: 2026-06-05 19:44:00 +0800
categories: [Unreal]
tags: [til,unreal]
render_with_liquid: false
---

## Rotation
### Get
- GetActorRotation()
  엑터 월드 기준 회전값 반환

- GetActorRelativeRotation()
  부모 컴포넌트 기준으로 상대적 회전값 반환

### Set
- SetActorRotation(FRtator)
  엑터 월드 기준 회전값 설정

- SetActorRelativeRotation(FRtator)
  액터의 상대 회전값을 설정

- AddActorWorldRotation(FRtator)
  현재 월드 회전값에 각도를 더해 회전

- AddActorLocalRotation(FRtator)
  액터의 로컬 회전값에 각도를 더해 회전



## Location
### Get
- GetActorLocation()
  액터의 월드 기준 위치 반환

- GetActorRelativeLocation()
  액터를 소유한 부모 컴포넌트 기준 상태 위치 반환

### Set
- SetActorLocation(FVector)
  액터 위치를 월드기준 위치로 변경

- SetActorRelativeLocation(FVector)
  액터의 상대 위치를 설정

- AddActorWorldOffset(FVector)
  현 월드 위치에 추가 위치값을 더하여 위치 설정

- AddActorLocalOffset(FVector)
  로컬 위치에 위치값을 더해 로컬 위치 설정

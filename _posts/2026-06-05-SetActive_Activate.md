---
title: "[Unreal]Component Active"
author: siha
date: 2026-06-05 19:44:00 +0800
categories: [Unreal]
tags: [til,unreal]
render_with_liquid: false
---

## SetActive(bool , bool bReset)
매개 변수에 따라
컴포넌트의 기능이 켜지고 꺼진다. 

## Activate(bool bReset)
컴포넌트를 활성화

## DeActivate()
컴포넌트를 비활성화

## SetVisibility(bool)
컴포넌트의 가시성을 활성화 / 비활성화

## ToggleVisibility()
현재 상태를 반전하는 함수

## SetCollisionEnabled(ECollisionEnabled)
* ECollidionEnabled 에 대해 추후 포스팅
컴포넌트의 충돌 상태를 열거형으로 제어

## SetCollisionProfileName(string)
미리 설정해 둔 콜리전 프로필 이름을 통해 설정

>bReset 매개변수는 true일 경우 파티클, 오디오, 타임라인 등 진행사항이 초기화되고 처음부터 다시 진행된다{: .prompt-tip}

## 알게 된 점
StaticMesh를 켜고 끄기 위해 SetActive를 사용하였다.
하지만 Visual이 꺼지진 않았다.

Static은 정적 오브젝트이고 SetActive로 비활성화 될 경우 Tick이 중지되기에 Visual에는 영향이 없게 된다.
차라리 Visual만 끄는 함수가 더 효율이 좋고 문제가 없다.

컴포넌트의 기능을 통재로 켜고 끄기에 부담이 상대적으로 크다.

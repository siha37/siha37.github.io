---
title: "[TIL]Unreal 엔진 빌드 프로세스"
author: siha
date: 2026-06-04 19:10:00 +0800
categories: [TIL]
tags: [til,unreal]
render_with_liquid: false
---

## 오늘 한 것


| 오늘 한 것                            |
| ------------------------------------- |
| Unreal Visual Studio 솔루션 구조 이해 |
| C++ Actor Class 생성                  |
| 채용공고 워크샵                       |


## [Unreal Visual Studio 솔루션 구조 이해](/posts/UnrealVisualStudioSolution)

## Live Coding
> Unreal 엔진이 실행되는 동안 C++ 코드를 수정하고 실시간 컴파일하여 결과물을 엔진에 반영할 수 있게 해주는 시스템

### 특징
- 엔진을 껐다 켰다하는 방식이 아닌 바로 적용되어 개발자의 번거러움과 작업 시간이 줄어든다.
- 복잡학 C++ 코드를 변경할 경우 적용되지 않는 경우도 존재한다.
- 빠른 적용을 위해 patch 파일 형태로 컴파일하는데 해당 파일은 바이너리에 영구 저장이 아니여서 빌드하지 않고 에디터를 종료하거나 비정상 종료되면 해당 작업 내용이 날아감

## [Actor Class 생성](/posts/UnrealActorClassCreat)


## 어려웠던 점
Unreal에서 지원하는 매크로와 기능들이 단순하게 사용하면 사용할 수 있지만 깊게 이해하기에는 많은 것을 파악해야 하는 어려움이 존재함

## 배운 점
### 배움
Unreal C++ 작업의 어려움과 불편함이 존재하고
이를 해결하기 위한 대응책이 필요하다 느낌

### 의미 
위와 같은 문제점은 지속적으로 경험하고 적응하는 방법이 옳다고 생각되며 C++의 Unreal 코드에 대해서 지속적인 공부가 필요하다 생각되는 시간이였음

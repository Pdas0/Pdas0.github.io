---
layout: post
title: "[Case-Study] JITSploitation II: Getting Read/Write"
date: 2025-02-21 20:53 +0800
tags: [Pwnable, Webhacking]
toc:  true
---

원본 글 : [GoogleProjectZero-JITSploitation II: Getting Read/Write](https://googleprojectzero.blogspot.com/2020/09/jitsploitation-two.html)
{: .message }

- 이전까지의의 내용은 1편을 참고해주시기 바랍니다.
[1편](https://pdas0.github.io/2025/02/20/Case-Study-JITSploitation-I-A-JIT-Bug/)

## Faking Objects

객체를 위조하기 위해선 해당 객체의 인메모리 레이아웃을 알아야합니다.

JSC의 일반 JSObject는 JSCell 헤더와 **버터플라이** 및 인라인 프로퍼티로 구성됩니다.

**버터플라이**는 객체의 프로퍼티와 요소, 요소의 길이를 포함하는 스토리지 버퍼입니다.

![Layout of a JSC Butterfly in memory]()

JSArrayBuffers와 같은 객체는 JSObject 레이아웃에 추가 멤버를 더합니다.

각 JSCell 헤더는 런타임의 StructurelDTable에 대한 인덱스인 JSCell 헤더의 StructureID 필드를 통해 구조체를 참조합니다.

해당 구조체는 다음과 같은 정보를 포함하는 유형의 정보를 가지고 있습니다.
- JSObject, JSArray, JSString, JSUint8Array와 같은 객체의 기본 유형
- 객체의 속성 및 객체를 기준으로 저장된 위치
- 바이트 단위의 객체 크기
- 버터플라이에 저장된 배열 요소(JSValue, Int32, Unboxed double)와 인접한 하나의 배열로 저장되는지 아니면 맵과 같은 다른 방식으로 저장되는지를 나타내는 인덱싱 유형
- 기타 등등.

마지막으로, 나머지 JSCell 헤더 비트는 GC 마킹 상태와 같은 것을 포함하고 인덱싱 유형과 같이 자주 사용되는 유형 정보 중 일부를 **캐시**합니다.

아래 이미지는 64비트 아키텍처에서 일반 JSObject의 인메모리 레이아웃을 정리한 것입니다.

![Layout of a JSC JSObject in memory]()

객체에 대해 수행되는 대부분의 작업은 객체의 구조를 보고 객체로 수행할 작업을 결정해야 합니다.

따라서 가짜 JSObject를 만들 때는 가짜로 만들려는 객체 유형의 StructureID를 알아야 합니다.

이전에는 StructureID Spraying을 사용하여 StructureID를 예측할 수 있었습니다.

이 방법은 원하는 유형의 객체(예: Uint8Array)를 여러 개 할당하고 각 객체에 다른 프로퍼티를 추가하여 고유한 구조와 그에 따라 해당 객체에 대한 StructureID를 할당하는 방식으로 작동하였습니다.

이 작업을 1000번 정도 수행하게 되면 1000이라는 숫자가 Uint8Array 객체에 대한 유효한 StructureID라는 것을 사실상 보장할 수 있었습니다.

바로 이 지점에서 2019년 초부터 적용된 새로운 보호 기법인 **StructureID Randomization**이 나타나게 됩니다.

## StructureID Randomization

이 보호 기법의 아이디어는 매우 간단합니다.


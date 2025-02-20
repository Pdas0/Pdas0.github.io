---
layout: post
title: "[Case-Study] JITSploitation I: A JIT Bug"
date: 2025-02-20 11:22 +0800
tags: [Pwnable, Webhacking]
toc:  true
---

참고 : [GoogleProjectZero-JITSploitation I: A JIT Bug](https://googleprojectzero.blogspot.com/2020/09/jitsploitation-one.html)

## JIT Compiler

JIT 컴파일러를 간단하게 정리해보면 다음과 같습니다.

다음과 같은 JavaScript 코드가 있습니다.
```js
function foo(o, y) {
    let x = o.x;
    return x + y;
}

for (let i = 0; i < 10000; i++) {
    foo({x: i}, 42);
}
```

JIT 컴파일러는 동작하는데 많은 시간이 소요되기 때문에, 일반적으로 반복적으로 실행되는 코드에 대해서만 수행이 됩니다.

따라서 위의 foo 함수는 JIT 컴파일러 없이 일정시간동안 인터프리터 단계에서 실행이 됩니다.

그리고 이 시간동안 foo 함수에 대해 프로파일 값이 수집되는데, foo 함수의 경우 다음과 같이 프로파일이 수집됩니다.

- o: JSObject, x 속성이 offset 16에 존재
- x: Int32
- y: Int32

이후 최적화를 위해 JIT 컴파일러가 활성화되면, JavaScript 소스 코드를 JIT 컴파일러 자체 IR(Intermediate Representation)로 변환하는 과정이 시작됩니다.

그리고 Case-Study에서 사용될 JavaScriptCore의 JIT 컴파일러인 DFG(Data Flow Graph)는 이 작업을 DFGByteCodeParser가 해당 과정을 수행하게 됩니다.

DFG의 IR 형태로 foo 함수는 초기에는 다음과 같이 변환될 수 있습니다.
```makefile
v0 = GetById o, .x
v1 = ValueAdd v0, y
Return v1
```

여기서 GetByID와 ValueAdd는 다양한 타입에 대해서도 연산이 가능한 함수이므로, 다양한 입력 타입을 처리할 수 있습니다.
(e.g. ValueAdd는 정수 덧셈뿐만 아니라 문자열 Concatenation도 수행)

위와 같이 변환된후 DFG는 수집된 값의 프로파일을 분석하고, 이를 기반으로 유사한 입력 타입이 계속 사용될 것이라고 추론을 합니다.

여기서는 o가 항상 특정 유형의 JSObject이며, x와 y가 Int32일 것이라고 예측합니다.

그러나 위와 같은 추론은 항상 맞는다는 보장이 없습니다. 컴파일러는 이를 보호하기 위해 런타임 단계에서 먼저 타입 체크를 수행하게 됩니다.

위와 같은 단계를 거친 후 생성된 최적화 IR는 다음과 같습니다.
```vbnet
CheckType o, "Object with property .x at offset 16"
CheckType y, Int32
v0 = GetByoffset o, 16
CheckType v0, Int32
v1 = ArithAdd v0, y
Return v1
```
- 위의 코드에서 CheckType은 해당 변수가 예상된 타입인지 확인하는 역할을 합니다.

- GetByoffset은 GetById보다 좀 더 최적화된 형태로, 특정 offset에서 직접 값을 가져옵니다.

- AritAdd는 ValueAdd보다 더 특화된 연산으로, 여기서는 Int32 덧셈만 수행합니다.

위와 같은 과정은 추론이 맞을 경우에는 실행 속도가 매우 향상되지만, 만약 가정이 틀릴 경우에는 일반적인 실행 경로로 복귀하여 정상적으로 동작 할 수 있도록 해줍니다.

또한 GetById와 ValueAdd 연산이 좀 더 효율적인 GetByOffset과 ArithAdd 연산으로 바뀌었다는 점을 주목해야합니다.

DFG는 이러한 추론적 최적화(speculative optimization)이 여러 단계에 거처서 수행이 됩니다.
(e.g. DFGByteCodeParser)

이 시점에서 IR 코드는 추론 보호(speculation guards)를 통해 타입 추론이 가능해지므로, 사실상 명확한 타입이 지정된 상태가 됩니다.

이후 루프 내부의 변하지 않는 코드를 루프 외부로 이동하는 LICM(Loop Invariant Code motion)이나 런타임 대신 컴파일 시간에 표현식을 평가하는 과정인 Constant Folding 기법과 같은 다양한 최적화 기법들을 수행하게 됩니다.

DFG에서 적용된 최적화 개요들은 DFGplan을 통해 확인할 수 있습니다.

최적화가 완료된 IR은 최종적으로 머신 코드로 변환되게 됩니다.
- 이 과정에서 DFG에서는 DFGSpeculativeJIT이 직접 IR을 머신 코드로 변환합니다.
- FTL 모드에서는 DFG IR이 먼저 B3라는 또 다른 IR로 변환된 후 추가 최적화 과정을 걸쳐 머신 코드로 변환됩니다.


## Common-Subexpression Elimination (CSE)

지금까지 설명한 최적화 기법의 핵심은 중복된 연산(또는 표현식)을 발견하고 이를 하나의 연산으로 병합하는 것입니다.

예를 들어 다음과 같은 코드가 있습니다.
```js
letc = Math.sqrt(a * a + a * a);
```

위 코드에서 a * a 는 두 번 계산되지만, 사실 동일한 연산이므로 이를 최적화할 수 있습니다.

JIT 컴파일러는 다음과 같은 중복 연산을 감지하고, 하나의 연산으로 합쳐 성능을 향상시킵니다.

또한 a와 b가 Number와 같은 primitive value라면, JS JIT 컴파일러는 다음과 같이 코드를 변환합니다.
```js
let tmp = a * a;
let c = Math.sqrt(tmp + tmp);
```

위의 최적화를 통해 ArithMul 연산을 두 번에서 한 번으로 줄일 수 있습니다.

위와 같은 최적화 기법을 공통 부분식 제거(Common Subexpression Elimination, CSE)라고 하며, 동일한 연산이 여러 번 수행되는 경우 이를 감지하여 한 번만 계산하고 재사용하는 방식으로 최적화를 진행합니다.

다음과 같은 코드를 살펴봅시다.
```js
let c = o.a;
f();
let d = o.a;
```

여기서 JIT 컴파일러는 CSE를 적용하여 두 번째 o.a의 로드를 제거할 수 없습니다.

그 이유는 중간에 호출되는 함수인 f()가 o.a의 값을 변경할 가능성이 있기 때문입니다.

따라서 let d = o.a;에서 이전에 동일한 값에 대해 사용이 있었음에도 불구하고, 다시 o.a 값에 대해 로드를 진행해야 합니다. 이러한 경우 **메모리 일관성(memory consistency)** 보장이 필요하며, JIT 컴파일러는 보수적인 접근 방식을 취해 최적화를 진행하지 않습니다.

JSC에서 특정 연산이 CSE의 대상이 될 수 있는지를 모델링하는 작업은 DFGClobberize에서 수행하게 됩니다.

이를 설명하기 위한 예시 코드는 다음과 같습니다.
```cpp
case ArithMul:
    switch (node->binaryUseKind()) {
    case Int32Use:
    case Int52RepUse:
    case DoubleRepUse:
        def(PureValue(node, node->arithMode()));
        return;
    case UntypedUse:
        clobberTop();
        return;
    default:
        DFG_CRASH(graph, node, "Bad use kind");
    }
```

위의 코드에서 `def(PureValue(node, node->arithMode()));` 부분은 해당 연산이 특정 컨텍스트에 의존하지 않으며, 동일한 입력에 대해 항상 돌일한 결과를 반환한다는 것을 의미합니다.

그러나 PureValue는 연산의 ArithMode를 매개변수로 받는데, 이는 해당 연산이 정수 오버플로우를 처리할지 여부를 지정한다.

이러한 매개변수화는 서로 다른 오버플로우 처리 방식을 가진 두 개의 ArithMul 연산이 동일한 연산으로 대체되는 것을 방지합니다.

오버플로우를 감지하는 연산은 일반적으로 **체크된(checked) 연산**, 오버플로우를 감지하지 않은 연산은 **체크되지 않은(unchecked) 연산**이라고 합니다.

반면, GetByOffset의 경우, DFGClobberize에서 다음과 같이 정의됩니다.
```cpp
case GetByOffset:
    unsigned identiferNumber = node->storageAccessData().identifierNumber;
    AbstarctHeap heap(NamedProperties, identifierNumber);
    read(Heap);
    def(HeapLocation(NamedPropertyLoc, heap, node->child2(), LazyNode(node)));
```

위의 코드를 살펴보면 NameProperty라는 Abstract Heap에 따라 값이 달라진다는 것을 알 수 있습니다.

따라서 CSE에 의해 두 번째로 호출되는 GetByOffset이 제거될려면, NamedPropertires 추상 힙에 대한 쓰기 연산이 없어야 합니다.

## The Bug

DFGClobberize에선 ArithNegate 연산 수행시 ArithMode를 고려하지 않았습니다.
```cpp
case ArithNegate:
    if (node->child1().useKind() == Int32Use || ...)
        def(PureValue(node));
```

위의 코드로 인해 **오버플로우가 체크된 ArithNegate 연산**이 **오버플로우가 체크되지 않은 ArithNegate 연산**으로 대체되는 문제가 발생할 수 있었습니다.

특히 ArithNegate에서 **정수 오버플로우가 발생할 수 있는 유일한 경우**는 INT_MIN을 부호 반전할 경우입니다.
- INT_MIN = -2147483648
- -INT_MIN = 2147483648 -> 32비트에선 표현 불가능 -> 오버플로우 발생
- -INT_MIN은 INT_MIN으로 다시 변환됨

정리하면, ArithNegate는 INT_MIN을 입력값으로 받는 경우 오버플로우 문제를 발생시킬 수 있다는 의미입니다.

위의 코드를 다시 살펴펴보면 ArithMode 없이 node를 PureValue로 처리했기 때문에 CSE가 ArithMode가 다른 ArithNegate 연산을 잘못 대처하는 문제가 발생할 수 있었습니다.

위의 코드에 대한 패치는 매우 간단합니다.
```diff
- def(PureValue(node));
- def(PureValue(node, node->arithMode()));
```

위와 같이 수정을 하면 CSE는 ArithNegate의 ArithMode를 확인하게 됩니다. 이를 통해 서로 다른 모드를 가진 두 ArithNegate 연산이 서로 대체되지 않도록 해줍니다.

DFGClobberize는 ArithNegate 뿐만 아니라 ArithAbs 연산에서도 ArithMode를 확인하지 않았습니다.

그리고 이러한 버그들은 퍼징으로 감지하기 매우 어렵습니다. 그 이유는 다음과 같습니다.
1. 퍼저가 동일한 입력값을 사용하는 두 개의 ArithNegate 연산을 생성해야 하지만, 각 연산이 서로 다른 ArithMode를 가져야 한다.
2. 퍼저는 ArithMode의 차이가 실제로 영향을 미치는 경우를 트리거 해야한다. 이 경우 INT_MIN 값을 부호 반전하는 상황을 만들어야 한다.
3. 퍼저가 이 조건을 **메모리 안전성 문제(memory safety violation) 또는 어썰션 실패(assertion failure)**로 이어지도록 만들어야 한다.

특히 단계를 퍼저만을 통해 이 조건을 우연히 충족하는 것은 매우 어렵습니다.


## Achieving Out-Of-Bounds

아래의 JS 코드는 위의 버그를 이용하여 임의의 인덱스를 통해 JSArray에 대한 **Out-Of-Bounds(OOB)**를 수행하는 코드입니다.
```js
function hax(arr, n) {
    n |= 0;
    if (n < 0) {
        let v = (-n) | 0;
        let i = Math.abs(n);
        if (i < arr.length) {
            if (i & 0x80000000) {
                i += -0x7ffffff9;
            }
            if (i > 0) {
                arr[i] = 1.04380972981885e-310;
            }
        }
    }
}
```

위의 코드를 차근차근 살펴보겠습니다.

먼저, ArithNegate는 정수를 대상으로만 부호 반전을 수행합니다. 하지만 JavaScript Specification에선 Number는 기본적으로 **부동소수점** 값으로 취급됩니다.

따라서 입력 값이 항상 정수임을 컴파일러에게 암시해줘야 합니다.

이를 쉽게 수행하는 방법은 비트 연산을 적용하는것인데, 비트 연산은 항상 32비트 부호 있는 정수값을 반환홥니다.

위의 코드에서 이 과정을 수행하는 코드는 다음과 같습니다.
```js
n = n | 0;
```

이제 **체크되지 않은** ArithNegate 연산을 생성할 수 있고, 이후 **체크된** 연산이 CSE에 의해 대채될 것입니다.
```js
n = n | 0;
let v = (-n) | 0;
```

여기서, **DFGFixupPhase**동안 n의 부호 반전 연산은 체크되지 않은 ArithNeg 연산으로 변환됩니다.

컴파일러는 부호 반전된 값이 오직 비트 OR 연산에만 사용됨을 확인하고, 이 연산이 오버플로우된 값과 정상적인 값에서 동일하게 동작하는 것을 확인할 수 있습니다.
```shell
js> -2147483648 | 0
-2147483648
js> 2147483648 | 0
-2147483648
```

다음으로는 n을 입력으로 하는 **체크된** ArithNegate 연산을 생성해야 합니다.

여기서 흥미로운 방법은 컴파일러가 ArithAbs 연산을 ArithNegate로 변환하도록 유도하는 것입니다.

이 최적화 방법은 컴파일러가 n이 항상 음수라는 것을 증명할 수 있을때만 발생합니다.

이는 DFG의 **IntegerRangeOptimization** pass가 path-sensitive이기 때문에 쉽게 달성할 수 있습니다.
```js
n = n | 0;
if (n < 0) {
    let v = (-n) | 0;
    let i = Math.abs(n);
}
```

여기서 바이트코드 파싱 중에 Math.abs 호출은 먼저 ArithAbs 연산으로 변환됩니다.

이는 컴파일러가 해당 호출이 항상 mathAbs 함수를 실행한다는 것을 증명할 수 있기 때문입니다.

따라서 ArithAbs 연산으로 대체되며, 이는 런타임 단계에선 동일한 의미를 가진다는 것을 의미합니다.

이를 통해 다음 실행시 더 이상의 함수 호출은 필요로 하지 않게 됩니다.

즉, **컴파일러가 Math.abs를 인라인(inlining)하는 효과를 가지게 됩니다.**

이후 IntegerRangeOptimization Path는 ArithAbs를 **체크된** ArithNegate로 변환합니다.
(이때 ArithNegate는 INT_MIN을 배제할 수 없으므로 반드시 체크된 연산이어야 합니다.)

결과적으로 if문 내부의 두 문장은 DFG IR의 pseudo DFG IR로 다음과 같이 변환됩니다.
```ini
v = ArithNeg(unchecked) n
i = ArithNeg(checked) n
```

이 버그로 인해 CSE는 다음과 같이 잘못된 변환을 수행하게 됩니다.
```ini
v = ArithNeg(checked) n
i = v
```

위의 최적화가 수행된 이후 부터는 n에 INT_MIN을 전달하여 잘못 컴파일된 함수를 호출하면, i는 양수가 되어야 하지만, INT_MIN이 저장되게 됩니다.

위의 버그 자체는 정확성 문제이지, 보안 문제는 아닙니다.

그러나 이 버그를 보안 취약점으로 악용할 수 있는 방법은 JIT 최적화 기법 중 하나인 **경계 검사 제거(bounds-check elimination)**을 악용하는 것입니다.

IntegerRangeOptimization Path로 돌아가 보면, i의 값은 이미 양수로 마킹되어 있습니다.

그러나 경계 검사 제거가 발생하려면, i가 배열의 길이보다 작은 값임이 보장되어야 합니다.

이를 다음 코드를 통해 충족시킬 수 있습니다.
```js
function hax(arr, n) {
    n = n | 0;
    if (n < 0) {
        let v = (-n) | 0;
        let i = Math.abs(n);
        if (i < arr.length) {
            arr[i];
        }
    }
}
```

위의 코드를 통해 버그가 트리거 되면, i는 INT_MIN이 되고, 배열 경계 검사를 통과한 후 배열에 접근하게 됩니다.

그러나 IntegerRangeOptimization은 i가 항상 배열 경계 내에 있다고 잘못 판단했기 때문에, 경계 검사가 제거된 상태에서 실행되게 됩니다.

버그를 트리거하기 위해선 JS 코드가 JIT 컴파일 되기 전에 충분히 실행되어야 합니다. 이는 단순히 반복적으로 실행해주면 됩니다.

하지만 arr에 대한 인덱스 접근이 **SSALoweringPhase**에서 CheckInBounds와 경계 검사가 없는 GetByVal로 변환되려면, JIT가 해당 접근이 항상 경계 내라고 추론해야 합니다.

이 말은 즉, 기본 JIT 실행 중 Out-of-Bounds 접근이 자주 감지되게 되면, JIT는 위의 최적화 과정을 수행하지 않습니다.

따라서 학습 과정에서 정상적인 인덱스만을 사용하여야 합니다.
```js
for (let i = i; i <= ITERATIONS; i++) {
    let n = -4;
    if (i == ITERATIONS) {
        n = -2147483648;
    }

    hax(arr, n);
}
```



 
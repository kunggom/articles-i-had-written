# AoS와 SoA, 그리고 DOD

<p align="right">큰곰(조진식, kunggom@gmail.com)<br />2019년 10월 22일</p>

흔히 들어보지 못했을 개념에 대해 잠시 정리하여 공유해보도록 하겠습니다. 오늘은 AoS와 SoA, 그리고 데이터 지향 디자인(DOD)에 대해 알아보겠습니다.

## 메모리 레이아웃

우리가 메모리에 뭔가 여러 가지 복합적인 정보를 모아서 저장할 때, 데이터 공간을 나누는 2가지 방법이 있습니다. 첫 번째는 데이터를 객체 단위로 나누어 모아두는 것이고, 두 번째는 데이터를 같은 종류끼리 모아두는 방식입니다. 첫 번째 방법을 AoS(Arrays of Structures)라고 하고, 두 번째를 SoA(Structures of Arrays)라고 부릅니다.

## AoS(Arrays of Structures)

말은 어렵지만, AoS는 우리에게 굉장히 익숙한 객체들을 한데 모아둔 것과 같다고 봐도 무방합니다.

```java
class Monster {
    String name;
    int hp;
    float attackpower;

    Monster(String name, int hp, float attackpower) {
        this.name = name;
        this.hp = hp;
        this.attackpower = attackpower;
    }
}

public static void main(String... args) {
    Monster[] monsters = new Monster[256];
}
```

위 코드를 보면 `Monster` 객체 안에는 서로 다른 형(Type)의 데이터 3개가 들어 있고, 이 객체가 모두 256개 들어갈 수 있는 배열이 준비되는 것을 확인할 수 있습니다. 이런 구조가 바로 AoS로, 우리에게는 아주 익숙한 형태입니다. 객체지향 패러다임에서는 필수적이지요.

## SoA(Structures of Arrays)

반대로, SoA는 좀 이질적인 느낌이 들 것입니다.

```java
class MonsterGroup {
    String[] name;
    int[] hp;
    float[] attackpower;

    MonsterGroup(String[] name, int[] hp, float[] attackpower) {
        this.name = name;
        this.hp = hp;
        this.attackpower = attackpower;
    }
}

public static void main(String... args) {
    int n = 256;
    MonsterGroup monsters = new MonsterGroup(
        new String[n],
        new int[n],
        new float[n]
    );
}
```

위 코드를 보면 `MonsterGroup`이라는 하나의 커다란 객체 안에 각 몬스터를 구성하는 정보들이 형(Type)에 따라 나뉘어 배열에 담겨 있는 것을 볼 수 있습니다. 그러니까 객체가 아니라 형(Type)에 따라 데이터를 구분하는 것이죠.

## 이런 걸 왜 쓰나?

얼핏 보기에 SoA는 굉장히 이상하고 어색한 구조로 보일 수 있습니다만, 엄연히 쓰이는 곳이 있습니다. 바로 **게임이나 시뮬레이션과 같이 수많은 객체가 동시에 사용될 때 성능 개선**을 위해 이런 구조를 사용합니다.

이런 구조가 성능을 개선할 수 있는 원리는 간단합니다. 제가 예전에 캐시(Cache)에 대해 발표했을 때 ‘순차 지역성’(Sequential Locality)에 대해 언급했던 것을 기억하시나요? 이러한 구조는 순차 지역성을 높여서 CPU 캐시의 효율을 최대한으로 끌어올릴 수 있습니다. 왜냐하면 같은 형태의 데이터가 메모리 공간에 순서대로 모여 있기 때문이지요.

![데이터 지역성의 원리](https://i.imgur.com/R8YoXRU.png)

게임을 예로 들어, 위 코드에서 플레이어 캐릭터가 광역 필살기를 써서 필드 내 모든 몬스터의 체력(HP)을 `-5`씩 깎는 경우를 생각해 보겠습니다. AoS를 쓰는 경우라면 메모리에서 CPU의 캐시로 미리 여러 개의 `Monster` 객체를 가져다 둬도 실제로 그 중에서 한번에 처리가 되는 데이터의 양은 얼마 되지 않습니다. (사실 Java의 경우에는 좀 더 복잡하지만, 일단은 이렇게 서술합니다.) 왜냐하면 `Monster` 객체에서 `hp`라는 필드는 여러 필드 중 하나일 뿐이니까요. 그런데 SoA에서는 메모리에서 CPU 캐시로 가져오는 모든 데이터가 `hp` 값을 나타내는 `int` 데이터이기 때문에 캐시 미스(Cache miss) 없이 최고 효율로 처리가 가능한 것이죠.

하드웨어 자체적으로도 SoA를 쓰는 추세가 있습니다. 예전에는 CPU나 GPU에서 AoS 기반으로 데이터를 처리하는 것이 대세였습니다. 서로 다른 데이터 여러 개를 하나로 묶은 단위(즉 AoS 방식)에 하나의 명령어를 적용하여 전체 데이터를 처리하는 방식을 가리켜 SIMD(Single Instruction Multiple Data)라고 합니다. 이 방법은 처리해야 할 데이터들이 정확하게 묶여 있다면 효율이 높지만, 그렇지 않을 때는 효율이 떨어진다는 단점이 있습니다. 예를 들어서 4가지 데이터를 묶어서 SIMD로 처리하는 하드웨어에 3가지 데이터만 묶어서 집어넣으면 처리 효율은 75%밖에 되지 않을 것입니다. 그러나 같은 종류의 데이터만을 모아서(=SoA) 각각 처리할 경우, 이런 식의 비효율성은 나타나지 않게 됩니다. 처리해야 할 데이터의 종류가 다양해진 지금은 CPU나 GPU 모두 병렬 데이터 처리를 위해 SoA를 사용하는 SIMT(Single Instruction Multiple Thread) 방식을 채택한 것이 대부분입니다.

![AoS vs. SoA](https://i.imgur.com/7O33vzF.jpg)

이 방식에도 단점은 있습니다. 가장 큰 것은 역시 데이터를 분해했다 다시 헤쳐모여하는 과정의 오버헤드가 있다는 점입니다. 이 때문에 처리해야 할 데이터의 양이 적으면 오히려 느려질 수도 있습니다. 하드웨어도 더 복잡해지게 되고, 무엇보다도 이를 다룰 프로그래머가 이런 구조에 익숙하지 않은 경우가 많습니다. 그럼에도 불구하고, 이런 구조는 동시에 처리해야 할 데이터의 양이 많으면 많아질수록 성능이 높아진다고 합니다.

## DOD(Data-Oriented Design)

제가 이해하기로, 데이터 지향 디자인(DOD, Data-Oriented Design)은 바로 SoA를 적극적으로 활용하는 디자인 방법론입니다. 지금까지 이 방법론은 그 뛰어난 처리 성능에도 불구하고 일부 게임에 사용된 것을 제외하면 그리 널리 쓰이지는 못했습니다. 무엇보다도 객체지향 방법론보다 코드를 알아보기 훨씬 어려울 뿐만 아니라, 기존의 객체지향 코드와 이 방법론을 연동하는 것도 까다롭기 때문입니다.

근래에 들어서 게임 업계를 중심으로 이 방법론을 좀 더 적극적으로 적용하려 하는 움직임이 있습니다. 유명 게임 엔진인 Unity에서 DOTS(Data-Oriented Technology Stack)라는 이름으로 DOD를 좀 더 쉽게 적용할 수 있는 시스템을 구축하고 있는 것이 대표적인 사례입니다. 이 시스템의 핵심은 ECS(Entity Component System)와 Job System인데, ECS란 바로 객체에서 연산에 필요한 필드만 모아 배열로 만들어주는 시스템이지요. 그리고 Job System은 이렇게 만들어진 배열에 실질적인 연산을 적용하는 부분입니다. 말하자면 DOTS는 객체지향과 데이터지향을 이어주는 가교 구실을 하는 셈입니다.

DOTS는 아직도 베타 상태로 활발히 개발 중이지만, 완성되고 나면 그동안 악명높았던 Unity의 느린 속도 문제를 확실히 개선할 수 있을 것으로 기대를 모으고 있습니다. 저는 이를 보면서 Java에도 추후 값 형식(Value Objects)이 도입되고 나면 Unity의 DOTS와 같은 것이 나올 수도 있겠다는 생각이 들더군요. 비록 DOD가 널리 쓰이지는 못할지라도, 처리해야 할 객체의 수가 엄청나게 많은 등 처리 효율이 아쉬운 상황에서라면 이런 개념 적용을 진지하게 고려해볼 만하지 않을까 합니다.

참고 :
 * https://blog.iwanhae.ga/intro-of-unity-dots/
---
categories:
  - Java
tags:
  - Java
published: true
created: 2024-04-13 (토) 02:07:44
modified: 2024-04-14 (일) 23:33:35
---

아래 자바 코드 실행 결과가 이해가 된다면 이 글을 스킵해도 된다.

```java
private boolean isBoxedIntegerSame(int i) {  
    Integer i1 = i;  
    Integer i2 = i;
    return i1 == i2;  
}

@Test
public void testIntegerSame(){
	System.out.println(isBoxedIntegerSame(127)); // true
	System.out.println(isBoxedIntegerSame(128)); // false
}
```

- isBoxedIntegerSame() 함수의 결과값이 true 혹은 false 중 하나의 값으로 예측했지만 틀렸다면, 자바의 동등 비교(\==)에 대한 이해나, Wrapper 타입의 boxing에 대한 이해가 부족했다고 생각하고 이 부분을 좀 더 찾아볼 수 있다.
- 그러나, 위 예시처럼 결과 값이 인풋에 따라서 달라지는 것은 위 두가지에 대한 이해만으로는 부족하다.
- 그래서 이 값이 달라지는 이유에 대해서 다루고자 한다.

## Autoboxing

우선 몇줄 안되는 isBoxedIntegerSame() 함수의 코드를 살펴보자.
```java
Integer i1 = i
```
와 같은 코드를 작성하게 되면, 컴파일러에서는 원시 타입인 int를 wrapper 타입인 Integer로 변환하기 위해 autoboxing을 수행한다.
이때 컴파일러는 `Integer.valueOf(int i)` 함수를 이용하게 된다.
그러므로 위와 같은 코드는 컴파일러에 의해
```java
Integer i1 = Integer.valueOf(i)
```

와 같은 코드로 변환된다.

그렇다면 Integer.valueOf() 함수의 내부 구현을 살펴보자

## Integer.valueOf 내부 구현

아래는 Integer.valueOf()의 구현 전문이다.
```java
/**  
 * Returns an {@code Integer} instance representing the specified  
 * {@code int} value.  If a new {@code Integer} instance is not  
 * required, this method should generally be used in preference to
 * the constructor {@link #Integer(int)}, as this method is likely
 * to yield significantly better space and time performance by 
 * caching frequently requested values. 
 * 
 * This method will always cache values in the range -128 to 127, 
 * inclusive, and may cache other values outside of this range. 
 * 
 * @param  i an {@code int} value.  
 * @return an {@code Integer} instance representing {@code i}.  
 * @since  1.5  
 */
 public static Integer valueOf(int i) {  
    if (i >= IntegerCache.low && i <= IntegerCache.high)  
        return IntegerCache.cache[i + (-IntegerCache.low)];  
    return new Integer(i);  
}
```
- [jdk/src/java.base/share/classes/java/lang/Integer.java at c1c99a669bb7f9928086db6a4ecfc90c410ffbb0 · openjdk/jdk · GitHub](https://github.com/openjdk/jdk/blob/c1c99a669bb7f9928086db6a4ecfc90c410ffbb0/src/java.base/share/classes/java/lang/Integer.java#L1016-L1020)

놀랍게도, 단순히 new Integer()를 호출하는게 아니고, 캐시의 개념이 들어가 있는 것을 확인할 수 있다!
구현을 보니, 특정 구간인 경우에 캐시에서 꺼내오고 그렇지 않으면 새로운 인스턴스를 생성하는 것으로 보인다.
이때 특정 구간이라함은, 위 자바독에 따르면 -128 ~ 127 (inclusive)에 해당한다.
캐시이기도 하고, 객체 재사용을 위해 사전에 미리 만들어놓은 풀(pool)의 개념이기도 하다.

이제 글 도입부의 실행결과를 이해할 수 있다.
127은 Integer 풀에 속한 구간이므로, Integer 인스턴스를 항상 풀에서 꺼내오기 때문에 늘 같은 인스턴스를 참조하게 되고. 그렇기 때문에 동등 연산을 수행하면 true가 나온다.
하지만, 128은 Integer 풀을 벗어난 구간이므로, 매번 새로운 인스턴스를 생성하게 되어 동등 연산을 수행하게 되면 false가 된다.

## JLS에 명시된 스펙

사실 특정 구간의 Integer에 대해 pool을 만들어두고 재사용하는 것은 시간과 공간 성능 상의 이유겠거니라고 생각했는데(자바의 인스턴스 생성은 비싼 연산이므로..)
이유는 성능 상의 이유가 맞지만, 관련해서 찾아보다보니 해당 구간의 Integer를 동일한 인스턴스를 참조하게 하는 것은 자바 언어의 스펙이었다.

JLS(Java Language Specification) 문서 중 Boxing conversion 파트에 아래와 같은 내용이 있다.
> If the value _p_ being boxed is `true`, `false`, a `byte`, a `char` in the range \u0000 to \u007f, or an `int` or `short` number between -128 and 127, then let _r1_ and _r2_ be the results of any two boxing conversions of _p._ It is always the case that _r1 == r2.

자바 버전별로 조금씩 표현은 다르지만, 결국 -128 ~ 127 구간의 Integer에 대해서는 Wrapper 타입이더라도, 동등 연산만으로도 값의 비교가 가능해야한다는 내용이다.
(그리고 Integer 외에도 Boolean, Character 등도 같은 개념이 있다.)

관련해서 조금 더 살펴보면, 아래와 같은 내용이 있다.
> Ideally, boxing a given primitive value p, would always yield an identical reference. In practice, this may not be feasible using existing implementation techniques. The rules above are a pragmatic compromise. The final clause above requires that certain common values always be boxed into indistinguishable objects. The implementation may cache these, lazily or eagerly.

결국 Wrapper 타입도 이상적으로는 모든 구간에 대해서 동등 연산만으로 비교가 가능해야하지만, 현실적으로 모든 수에 대해 객체를 만들어두는 것은 불가능하므로, 프로그래밍 상의 절충안으로 특정 구간으로 한정한다는 내용이다.

## AutoBoxCacheMax

위에서 Integer 풀의 구간은 -128~127 이라고 했는데, 해당 구간은 내부적으로 IntegerCache의 low, high 변수에 의해 조정된다.
Integer 클래스 내에 있는 IntegerCache 클래스의 내부 구현을 살펴보자.

```java
/**  
 * Cache to support the object identity semantics of autoboxing for values between 
 * -128 and 127 (inclusive) as required by JLS. 
 * 
 * The cache is initialized on first usage.  The size of the cache 
 * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.  
 * During VM initialization, java.lang.Integer.IntegerCache.high property 
 * may be set and saved in the private system properties in the 
 * sun.misc.VM class. 
 * 
 */  
private static class IntegerCache {  
    static final int low = -128;  
    static final int high;  
    static final Integer cache[];  
  
    static {  
        // high value may be configured by property  
        int h = 127;  
        String integerCacheHighPropValue =  
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");  
        if (integerCacheHighPropValue != null) {  
            try {  
                int i = parseInt(integerCacheHighPropValue);  
                i = Math.max(i, 127);  
                // Maximum array size is Integer.MAX_VALUE  
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);  
            } catch( NumberFormatException nfe) {  
                // If the property cannot be parsed into an int, ignore it.  
            }  
        }  
        high = h;  
  
        cache = new Integer[(high - low) + 1];  
        int j = low;  
        for(int k = 0; k < cache.length; k++)  
            cache[k] = new Integer(j++);  
  
        // range [-128, 127] must be interned (JLS7 5.1.7)  
        assert IntegerCache.high >= 127;  
    }  
  
    private IntegerCache() {}  
}
```
- [jdk/src/java.base/share/classes/java/lang/Integer.java at c1c99a669bb7f9928086db6a4ecfc90c410ffbb0 · openjdk/jdk · GitHub](https://github.com/openjdk/jdk/blob/c1c99a669bb7f9928086db6a4ecfc90c410ffbb0/src/java.base/share/classes/java/lang/Integer.java#L938-L998)

이때 low 값은 -128로 고정이지만, high 값은 `-XX:AutoBoxCacheMax=<size>` 를 VM 옵션으로 지정해주면 조절이 가능하다.

실제로, 아래와 같이 Intellij에서 VM 옵션으로 최댓값을 1000으로 설정하고
![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/68078797-d8d6-4c19-875a-e99766626456)

글 도입부의 아래 함수를 실행하면, 둘다 true를 리턴하는 것을 확인할 수 있다.

```java
@Test
public void testIntegerSame(){
	System.out.println(isBoxedIntegerSame(127)); // true
	System.out.println(isBoxedIntegerSame(128)); // true
}
```

그리고 위 IntegerCache의 내부구현을 자세히 살펴보았다면 알겠지만, 해당 상한 값을 127 미만으로 설정하게 되면, 무시하고 상한을 127로 적용한다. (JLS에 명시된 스펙이기 때문이다.)
따라서, 해당 옵션을 100으로 지정하더라도 위 함수의 리턴값이 둘다 false로 바뀌진 않는다.

## 활용 방안에 대한 개인적인 생각

사실 실무에서 이걸 활용할 일은 많을 것 같진 않다. (활용 사례가 있다면 댓글에 알려주시면 감사하겠습니다.)
굳이 찾자면, 인풋 구간이 -128~127 내로 한정된 함수에 대해 극한의 최적화를 하는데 활용할 수 있을 듯하다.

그리고 이걸 활용했을때 문제가 될 만한 상황은 Integer 클래스에 대해 동등 비교 연산자(\==)를 썼을때 예상치 못한 동작을 하는 것이 문제일 것이다.
예를 들면, 테스트코드에서는 -128~127 구간의 수로 테스트하여 테스트를 통과하였으나, 실제 비즈니스 코드 상에서는 그외의 구간의 인풋이 들어와서 테스트 코드에서 기대하던 결과와 다른 결과가 발생한다면, 실제로 문제가 될 수 있다.

하지만, 개인적인 생각으로는 해당 문제가 발생하기 이전에 참조 타입인 Integer 클래스에 대해 equals()가 아닌 \==를 사용하여 동등비교를 하는 것 자체가 바람직하지 않은 듯하다.
물론, 해당 연산을 -127~128 구간에 대해서만 수행하는 것이 보장되어 있다고 하더라도, 비즈니스 로직 상에서 그정도의 최적화까지 필요로 하는 일이 많지 않을 듯 하다.
오히려 위와 같은 스펙을 모르는 개발자가 봤을때는 == 연산자를 쓰는 것을 오류라고 생각할 수 있다.

그리고 자바 스펙에 의존적인 코드이기 때문에, 현재까지의 최신의 버전에서는 잘 동작하겠지만, 이후의 버전에서도 잘 동작한다는 보장이 없다.

그럼에도 최적화가 필요하다면, 위와 같은 자바 스펙 상의 Integer 캐시 개념에 대한 주석정도는 필요할 것 같다.

## 결론

- 자바의 Integer 클래스는 -127 ~ 128 구간에 대해 풀을 만들어놓고 객체를 재사용한다.
- 그리고 이것은 JLS에 명시된 자바의 스펙이다.
- 이 사실을 활용하면 최적화는 되겠지만, 모르는 사람이 보면 의아할 수 있으니 주석과 함께 사용할 필요는 있어보인다.
- 결국 실제 활용보다는 pool의 개념, 혹은 디자인 패턴 중 [Flyweight 패턴]([Flyweight pattern - Wikipedia](https://en.wikipedia.org/wiki/Flyweight_pattern)) 활용 사례 중 하나 정도로 알고 있으면 좋을 듯하다.

## References

- [jdk/src/java.base/share/classes/java/lang/Integer.java at master · openjdk/jdk · GitHub](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/lang/Integer.java)
- [Chapter 5. Conversions and Promotions](https://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.7)
- [java - Why Integer class caching values in the range -128 to 127? - Stack Overflow](https://stackoverflow.com/questions/20897020/why-integer-class-caching-values-in-the-range-128-to-127)
- [Flyweight pattern - Wikipedia](https://en.wikipedia.org/wiki/Flyweight_pattern)

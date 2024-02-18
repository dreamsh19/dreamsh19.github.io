---
title: Java Optional Deep Dive
categories:
  - Java
tags:
  - Java
published: true
created: 2024-02-18 (일) 23:14:43
modified: 2024-02-18 (일) 23:54:06
---

자바에서 Optional을 사용하면서, Optional의 설계 철학과 사용법을 제대로 알고 사용하고 싶어서 자세히 찾아보게 되었다.

## 1. Optional은 왜 등장했는가?

자바 개발자라면 모를수가 없는 악명 높은 예외가 바로 NullPointerException(이하 NPE)이다.
널 참조는 [billion dollar mistake](https://en.wikipedia.org/wiki/Null_pointer#History) 라고 불릴만큼, 악명 높은 디자인 실수(?)인데, 그 이유는 개발자의 실수에 의해 error-prone하기 때문이지 않을까 싶다.
Optional은 이러한 의도치않은 NPE를 방지하고자 등장했는데, 여러가지 자료를 찾아보고 개인적으로 정리한 Optional의 등장 배경은 크게 아래 두가지인 듯하다.

### API 디자인의 명확성 확보 및 NPE 방지

Optional의 설계의도와 관련하여 자바 아키텍트 Brian Goetz가 한 말이라고 한다.
>Optional is intended to provide a limited mechanism for library method return types where there needed to be a clear way to represent “no result," and using null for such was overwhelmingly likely to cause errors.

개인적으로 이 말이 Optional 설계의도를 가장 잘 드러내는 말이라고 생각한다.
저기서 주목한 점은 Optional의 애초에 "메소드의 리턴 타입"을 위해 설계되었다는 점이다. (후술할 Optional의 잘못된 사용과 연관되어 있는 부분이기도 하다.)

Optional이 없던 시절에는 메소드 리턴을 "결과 없음"을 표현하기 위해서는 null을 리턴하는 경우가 종종 있었고,
null을 리턴할 수 있다는 것 자체가 caller 쪽에 예외처리의 책임을 전가하는 것이기 때문에 NPE에 취약할 수 밖에 없다.

하지만 메서드가 Optional을 리턴한다면, caller 입장에서는 두 가지가 달라진다.
**첫번째로, 메서드 결과가 empty가 될 수 있음을 명시적으로 알 수 있고,**
**두번째로, 어쨌든 값을 꺼내서 쓰기 위해서는 Optional을 unwrap하는 과정이 강제되기 때문에, 기존의 널 체크와 유사한 로직이 강제된다.**

오라클의 Optional 관련 글 중에 Optional의 의도를 잘 이해할 수 있는 부분이 있어 발췌하였다.
> It is important to note that the intention of the `Optional` class is not to replace every single null reference. Instead, its purpose is to help design more-comprehensible APIs so that by just reading the signature of a method, you can tell whether you can expect an optional value. This forces you to actively unwrap an `Optional` to deal with the absence of a value.

### 코드의 가독성

```java
String version = computer.getSoundcard().getUSB().getVersion();
```

위와 같은 코드가 있다고 가정해보자. 비즈니스 로직 상에서 흔하게 볼 수 있는 로직의 형태이다.
그러나, null-safe한 코드 작성을 위해서는 아래와 같이 작성해야한다.
```java
String version = "UNKNOWN";
if(computer != null){
  Soundcard soundcard = computer.getSoundcard();
  if(soundcard != null){
    USB usb = soundcard.getUSB();
    if(usb != null){
      version = usb.getVersion();
    }
  }
}
```
더 길어진다면, 코드에 점점 indentation이 길어지고, 코드의 가독성은 점점 더 떨어질수 밖에 없다.
하지만 Optional은 메소드 체이닝을 지원하여 위와 같은 코드를 아래와 같이 작성할 수 있게 된다.

```java
String version = computer.flatMap(Computer::getSoundcard)
                          .flatMap(Soundcard::getUSB)
                          .map(USB::getVersion)
                          .orElse("UNKNOWN");
```

## 2. Optional의 생성

Optional 클래스의 프로퍼티는 아래와 같이 구현되어있다.
```java
public final class Optional<T> {  

	private static final Optional<?> EMPTY = new Optional<>();  

	private final T value;

}
```

Optional은 기본적으로 컨테이너일 뿐이기 때문에, 프로퍼티로 담고 있을 레퍼런스 하나가 전부이다.
(스태틱 필드로 `EMPTY` 인스턴스가 있긴 하지만, Optional의 의도와는 별개로 싱글턴 객체를 공유함으로써 시간 및 공간상 효율을 위한 프로퍼티이다.)

그리고 Optional의 생성자는 아래 단 두개 뿐이며, 둘다 private이다.
```java
private Optional() {  
	this.value = null;  
}

private Optional(T value) {  
	this.value = Objects.requireNonNull(value);  
}
```
이는 곧, 직접적인 생성자 호출을 통한 생성을 금지하려는 의도이다. 그렇다면, 실제 Optional 인스턴스를 생성하기 위해서는 어떻게 해야할까?

이를 위해서 Optional 클래스에서는 아래 세가지 static 메소드를 제공한다.
```java
public static <T> Optional<T> of(T value) {  
	return new Optional<>(value);  
}

public static<T> Optional<T> empty() {  
	@SuppressWarnings("unchecked")  
	Optional<T> t = (Optional<T>) EMPTY;  
	return t;  
}

public static <T> Optional<T> ofNullable(T value) {  
	return value == null ? empty() : of(value);  
}
```

## 3. 올바른 사용법
- map, filter 등의 각종 메소드는 사실 직관적이기 때문에 따로 정리할 필요는 없을 듯하고,
- 다만, 잘못 사용하는 경우 주의가 필요하여 정리하고자 한다.

### 3.1. 함수 파라미터로 Optional 사용

이는 사실 앞에서 기술한 Optional의 등장 배경과 철학을 알지 못하는 경우에 사용하기 쉽다.
아래와 같이 파라미터로 Optional을 받는 메서드(또는 생성자)가 있다고 가정해보자.
얼핏보면, Optional 이라는 이름에 걸맞게 attachment가 있는 경우와 없는 경우를 잘 표현한 듯 보인다.
```java
public SystemMessage(String title, String content, Optional<Attachment> attachment) {
	// 생략
	attachment.ifPresent(System.out::println);
}
```

그러나, 이는 Optional의 의도상 지양하는 코드이다.
그 이유는 위와 같은 메서드는 아래와 같은 코드를 호출가능하게 만들고, 이는 결국 런타임에 NPE가 발생할 수 밖에 없다.
```java
SystemMessage("title", "content", null)  
```

NPE를 방지하기 위해 만든 Optional을 잘못 사용하여 오히려 NPE가 발생하는 꼴이다.

### 3.2. 직렬화

Optional은 Serializable 인터페이스를 구현하지 않는다. 즉, 직렬화를 하려고 하면 NotSerializableException이 발생한다.
이 역시 사람들이 "null이 될 수 있는"의 의미로 Optional을 사용하면서 발생한 오해(?)이다. 사람들은 Optional을 클래스 내에서 "null이 될 수 있는 프로퍼티"로 사용하려고 했고, 그러면서 자연스럽게 직렬화에 대한 니즈가 발생했다.
그럼에도 현재까지 Optional에 Serializable 구현을 추가하지 않았는데, 그 이유는 Optional을 "메서드의 리턴타입"으로만 사용하게 하는 것이 본래 그 의도에 맞다고 판단했기 때문이라고한다.
- [Shouldn't Optional be Serializable?](https://mail.openjdk.org/pipermail/jdk8-dev/2013-September/003274.html)

### 3.3. 불필요한 orElse() 사용

Optional는 orElse()와 orElseGet() 두가지를 제공한다. 얼핏보면 비슷해보이는데, 두 가지의 차이는 무엇일까?

두 가지의 차이점은 "lazy하게 동작하는가"에 차이가 있다. 구현을 보면 알 수 있다.

```java
public T orElse(T other) {  
	return value != null ? value : other;  
}

public T orElseGet(Supplier<? extends T> other) {  
	return value != null ? value : other.get();  
}
```

orElseGet()은 Optional이 null이 아닐때만 lazy하게 결과를 가져온다.
하지만, orElse()는 함수 호출 이전에 이미 파라미터로 값을 넣어주어야하기 때문에 null 여부와 무관하게 항상 연산이 발생하게 되고, null이 아닌 경우에는 값이 쓰이지도 않기 때문에 이는 낭비이다.
만약 그 연산의 비용이 비싼 연산이라면(예를 들면, db와 통신 등) 불필요한 비용을 계속 발생시키게 된다.

## 4. 한계

개인적으로 생각한 Optional의 한계는 아래 두 가지인듯하다.
1. 시간과 공간상 오버헤드가 발생한다
	- 아무래도, wrap/unwrap 하는 과정이 필요하고 추가적인 메모리할당이 필요하기 때문에 오버헤드는 불가피해보인다.
2. 완전히 강제할 수 없다.
	- 개발자가 위의 "사용법"을 유념하고 사용해야한다는 것 자체가 한계로 보인다.
	- 기존에 null을 사용하던 것에 비해서는 훨씬 에러 가능성이 줄어들었지만, 여전히 `Optional.ofNullable(null).get()` 과 같은 형태의 코드를 컴파일 타임에 잡아낼 수 없다.
	- 이는 결국 Optional이 자바 언어 레벨에서 지원하는 것이 아니라 단순 코드 레벨에서 제공하기 때문에 발생할 수 없는 한계가 아닐까 생각한다.

## References

- [Tired of Null Pointer Exceptions? Consider Using Java SE 8's Optional!](https://www.oracle.com/technical-resources/articles/java/java8-optional.html)
- [Null safety - Kotlin Documentation](https://kotlinlang.org/docs/null-safety.html#nullable-types-and-non-nullable-types)
- [Optional (Java Platform SE 8 )](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html)
- [serialization - Why java.util.Optional is not Serializable, how to serialize the object with such fields - Stack Overflow](https://stackoverflow.com/questions/24547673/why-java-util-optional-is-not-serializable-how-to-serialize-the-object-with-suc)
- [\[Java\] 언제 Optional을 사용해야 하는가? 올바른 Optional 사용법 가이드 - (2/2) - MangKyu's Diary](https://mangkyu.tistory.com/203)
- [Guide To Java 8 Optional - Baeldung](https://www.baeldung.com/java-optional)
- [Shouldn't Optional be Serializable?](https://mail.openjdk.org/pipermail/jdk8-dev/2013-September/003274.html)

---
title: JsonInclude를 활용한 데이터 사이즈 축소
categories:
  - Java
tags:
  - Java
published: true
created: 2024-02-04 (일) 01:32:35
modified: 2024-02-04 (일) 23:57:42
---

## 이슈

- 서버에서 요청마다 남기는 로그 중 json으로 남기는 로그가 있었고, 서버에서 처리하는 요청수가 늘어남에 따라 로그 크기가 커지면서, 로그 크기를 줄여야하는 상황이 발생했다.
- 로그를 살펴보다보니, `"key":"value"` 를 구조로 된 json 포맷 특성상 의미없는 value(null, empty string, 빈 배열 등)를 갖는 필드에 대해서도 모두 키값이 기록되고 있음을 확인했고, 그 비율이 적지 않음을 확인했다.
- value가 의미없는 경우 필드 자체를 남기지 않게 되면 로그 크기가 많이 줄어들 것으로 예상되어 작업에 착수했다.
- 다만, 필드별로 "의미없음"의 의미가 다를 수 있기 때문에 각 필드별로 "의미없음"의 기준에 따라 json에 포함하지 않는 것이 필요했다.

## 현황

일반적으로 jvm 계열 언어(자바, 코틀린, 스칼라 등)에서 데이터 직렬화/역직렬화에 많이 사용하는 jackson 라이브러리를 사용해서 POJO 객체(아래 예시에서는 `LogEntry`)를 json으로 직렬화하고 있었다.
기존 코드 상에는 POJO 클래스 전체에 대해 `JsonInclude.Include.NON_NULL` 정도만 적용이 되어 있었다.
```java
@JsonInclude(JsonInclude.Include.NON_NULL)  
public class LogEntry {
	// 생략
	private final String name;
	// 생략
}
```

그렇기 때문에 필드별로 세분화된 `JsonInclude` 정책은 따로 없었고, 필드별 세분화된 설정을 위해서 코드 상에서 null로 바꿔주는 식으로 구현이 되어있었다.
예를 들어, String 타입의 특정 필드가 empty string("")인 경우 포함하지 않는 요구사항을 위해서 아래와 같이 null로 변환하여 넣어주는 식이었다.

```java
this.name = "".equals(name)? null : name;
```

위와 같은 코드는 비즈니스 로직이라기보다 단순히 직렬화에 포함시키지 않기 위한 목적의 코드라고 생각이 들었고,
코드의 가독성 측면에서도 `JsonInclude`라는 명시적인 애너테이션을 사용하는 것이 self-documenting 관점에서 더 직관적일 것 같다는 생각이 들었다.

그래서 JsonInclude의 구현을 찾아 들어가보니 `NON_NULL` 외에도 활용이 가능한 여러가지 enum이 있다는 것을 알게 됐다.
이 enum을 활용한다면, 코드 상에서의 별도 처리 없이 필드별 json 직렬화 기준을 오버라이딩할 수 있을 것 같다는 생각에 관련된 java doc을 자세히 살펴보았다.

## JsonInclude enum 종류

JsonInclude안에 들어갈 수 있는 값들은 기본적으로 "json에 어디까지 포함할지"에 그 범위에 대한 enum 타입들이며, 직렬화 단계에서만 적용되는 애너테이션이다.
(처음에는 역직렬화 단계에서는 어떻게 적용이 되는걸까 생각했으나, 역직렬화 단계에서는 사실 필요가 없다. 필드가 없으면 역직렬화 단계에서 해당 프로퍼티 타입의 디폴트 값이 될 것이고, 필드가 있다면 그대로 역직렬화를 하면 되기 때문이다. 역직렬화시 관련된 애너테이션에는 `@JsonIgnoreProperties`가 있다.)

아래에 jackson 라이브러리의 현재 기준 latest인 2.16.1 버전 기준으로, 사용가능한 JsonInclude의 enum 타입들에 대한 내용을 정리했다.
아래로 갈수록 json에서 제외되는 기준이 확장된다(혹은 포함될수 있는 기준이 타이트해진다)고 볼 수 있다.

### 1. ALWAYS
- 무조건 json에 포함한다.
- 별도로 지정하지 않았을때의 디폴트값이다.
	- [jackson-annotations/src/main/java/com/fasterxml/jackson/annotation/JsonInclude.java at e1910b8b7bfda325d450a30f1a121b5cf75d819a · FasterXML/jackson-annotations · GitHub](https://github.com/FasterXML/jackson-annotations/blob/e1910b8b7bfda325d450a30f1a121b5cf75d819a/src/main/java/com/fasterxml/jackson/annotation/JsonInclude.java#L58)

### 2. NON_NULL
- 명시적인 null일때만 제외
- 당연하게도, null이 될 수 없는 원시타입에 대해서는 ALWAYS와 동일하게 동작한다.

### 3. NON_ABSENT
- `NON_NULL` 기준을 상속하며,
- 컨테이너 타입(Optional, AtomicReference 등)의 내부가 비어있을 때 제외한다.
	+ 예를 들면, Optional의 isEmpty()인 경우 제외
- 주로 Optional 타입과 함께 사용한다.
- 2.6부터 사용 가능

### 4. NON_EMPTY
- `NON_ABSENT` 기준을 상속하며,
- 컬렉션, 배열, String이 empty인 경우 제외한다.
	- Collection, Map의 isEmpty 인 경우
	- 자바 배열 `length==0`
	- empty String("")
- 단, jackson 2.6에 대해서는 예외사항이 있음
	- 2.6 버전에서는 원시타입 등에 대해 empty의 범위가 확장되었으나(예를 들면 int의 0도 empty로 포함하였으나), 2.7 버전에서 롤백되었음.
	- 2.6을 제외한 버전에서는 위와 같은 경우 NON_DEFAULT를 사용

### 5. NON_DEFAULT
- 클래스(POJO)에 적용된 경우와 그렇지 않은 경우로 동작이 나뉜다.
- 클래스에 적용된 경우
	- 디폴트 POJO 객체(Zero Argument 생성자에 의해 생성된 객체)와 equals() 연산결과가 true를 리턴하는 경우 제외
- 클래스 외에 적용된 경우(global 설정 또는 프로퍼티에 적용된 경우)
	- `NON_EMPTY` 기준을 상속하며,
	- 원시타입 및 Wrapper 타입의 디폴트 값(ex. int의 0)인 경우 제외.
	- Date/Time 중 0L에 대응되는 경우 제외.

### 6. USE_DEFAULTS
- 실제 동작에는 영향을 주지 않고, Inclusion에 대한 디폴트 설정을 따르겠다는 것을 명시적으로 설정할때 사용한다.
- 즉, 명시하지 않은 것과 동일하게 동작한다. (명시하지 않아도 디폴트를 따를테니)
- 그럼에도 해당 값이 필요한 이유는 해당 값을 사용하게 되면, 디폴트 사용을 강제할 수 있기 때문에 다른 개발자가 inclusion 설정을 오버라이딩하는 것을 방지할 수 있다.
- 그리고 디폴트를 사용하겠다는 것을 명시적으로 선언하는 것이므로, 문서화로서의 의미도 있다.

## 실제 적용

위 내용을 바탕으로, 실제로는 아래 세 가지를 적용했다.
1. `NON_NULL`
	- 대부분의 경우 NON_NULL로 충분했기 때문에, (명시적으로 null인 경우가 실제로 로그에도 남길 필요 없는 경우였다.)
	- **클래스 단위에 주로 적용했다.**
	- 클래스 단위에 적용하는 경우, 해당 클래스 내 모든 프로퍼티는 그 속성을 따라가기 때문에 클래스 내 디폴트 설정으로 사용하기 적합했다.
	- 그리고 그 외에 추가로 직렬화 조건이 필요한 프로퍼티의 경우 아래 두가지로 오버라이딩 하는 방식을 채택했다.
2. `NON_EMPTY`
	- 로그 내 List 타입의 필드가 있었는데, empty List 인 경우에는 남기지 않기 위해 적용했다.
	- 오히려 String 타입의 프로퍼티의 경우, empty string("")인 경우에 명시적으로 남겨야하는 경우가 많아서 `NON_EMPTY` 대신 `NON_NULL`을 적용했다.
3. `NON_DEFAULT`
	- 원시 타입(int)에 대해 0인 경우에는 남기지 않도록 하기 위해 적용했다.
	- 참고로, 0과 null을 구분해야하고, 0인 경우 로그에 남아야하는 경우에는 wrapper type + `NON_NULL`의 조합을 채택했다.

실제 적용한 최종 결과를 요약하면 아래와 같은 형태이다.

```java
@JsonInclude(JsonInclude.Include.NON_NULL)  
public class LogEntry {

	@JsonProperty("tid")  
	private final String transactionId;  

	@JsonProperty("price")  
	private final Double price;  // 0.0과 null을 구분하고, 0.0은 로그에 남겨야하는 경우

	@JsonInclude(JsonInclude.Include.NON_DEFAULT)  
	@JsonProperty("et")  
	private final int elapsedTime;

	@JsonInclude(JsonInclude.Include.NON_EMPTY)
	@JsonProperty("res_list")
	private final List<Response> responseList;
}
```

그리고 위 내용을 적용 결과, 해당 설정만으로 로그 사이즈를 기존대비 25%나 줄일 수 있었다!

## References

- [GitHub - FasterXML/jackson-annotations at jackson-annotations-2.16.1](https://github.com/FasterXML/jackson-annotations/tree/jackson-annotations-2.16.1)
- [https://www.javadoc.io/doc/com.fasterxml.jackson.core/jackson-annotations/2.16.1/index.html](https://www.javadoc.io/doc/com.fasterxml.jackson.core/jackson-annotations/2.16.1/index.html)
- [\[SpringMVC\] 업무에서 활용한 @JsonInclude 사용법 정리](https://yuma1029.tistory.com/9)

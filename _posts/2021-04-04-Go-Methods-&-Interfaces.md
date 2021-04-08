---
categories:
  - Go
tags:
  - Go
---

## Method

- Go는 클래스가 없음

- 대신 함수에 Receiver를 넣어 메소드 작성 가능

- type `Vertex`가 선언된 패키지 내에서만 메소드 정의 가능(java 클래스 파일 내에서 관련 메소드 모두 정의하는 것과 유사)

- Receiver(객체) 원본에 접근해서 값을 변경하고자 하면 Pointer Recevier 사용 (C++ 참조 전달과 유사)

  - Pointer receiver를 사용하지 않으면 객체를 복사하여 값을 변경(함수 argument 넘겨주는 것처럼)

- ```go
  type Vertex struct {
  	X, Y float64
  }
  
  // Value receiver
  func (v Vertex) Abs() float64 { // v Vertex : receiver
  	return math.Sqrt(v.X*v.X + v.Y*v.Y)
  }
  
  // Pointer receiver
  func (v *Vertex) Scale(f float64) {
  	v.X = v.X * f
  	v.Y = v.Y * f
  }
  
  func main() {
  	v := Vertex{3, 4}
  	fmt.Println(v.Abs()) // 5
  	v.Scale(10) // == (&v).Scale(10). Pointer indirection(1)
  	(&v).Abs() // == v.Abs(). Pointer indirection(2)
  	fmt.Println(v.Abs()) // 50 (pointer receiver를 사용하지 않으면 5)
  }
  ```
  
-  Pointer indirection
  - (1) v는 value이고 Scale은 pointer receiver를 갖지만 Compile error 나지 않음. Go interpreter가 v.Scale()을 알아서 (&v).Scale()로 바꿔서 해석하기 때문
  - (2) 반대방향도 가능. (&v).Abs()를 알아서 v.Abs()로 바꿔서 해석
  - 단, 메소드(Receiver가  있는 함수)의 receiver에 대해서만 가능. 함수 argument에서는 pointer indirection 일어나지 않음
- Pointer Receiver를 사용하는 이유
  - 1. 직접 receiver를 수정할 수 있다.
  - 2. 메소드 호출마다 발생하는 복사를 방지할 수 있다. 
- **일반적으로 하나의 type에 대한 메소드는 pointer receiver만 갖거나 value receiver만 갖도록 한다. 둘이 섞어 쓰지 않는다.**
  
  - 이유는 ? 인터페이스에 할당 시에 통일성이 떨어져 활용이 번거로워진다.
  - 인터페이스에 포인터로 할당하면 pointer receiver를 갖는 메소드만 사용이 가능하다. Value receiver도 vice versa



## Interface

- 메소드의 집합

- ```go
  package main
  
  import (
  	"fmt"
  	"math"
  )
  
  type Abser interface {
  	Abs() float64
  }
  
  func main() {
  	var a Abser
  	f := MyFloat(-math.Sqrt2)
  	v := Vertex{3, 4}
  
  	a = f  // a MyFloat implements Abser
  	fmt.Println(a.Abs()) // 1.4142135623730951
  
  	a = &v // a *Vertex implements Abser
  	fmt.Println(a.Abs()) // 5
  
   	a = v	// Compile error : cannot use v (type Vertex) as type Abser in assignment:
  			// Vertex does not implement Abser (Abs method has pointer receiver)
  
  }
  
  type MyFloat float64
  
  func (f MyFloat) Abs() float64 {
  	if f < 0 {
  		return float64(-f)
  	}
  	return float64(f)
  }
  
  type Vertex struct {
  	X, Y float64
  }
  
  func (v *Vertex) Abs() float64 {
  	return math.Sqrt(v.X*v.X + v.Y*v.Y)
  }
  ```
  
- **명시적인 `implement` 표시 없이 해당 메소드(위의 예에서 `Abs()`)를 구현하면 interface를 implement했다고 판단.**

- 이 때 `Abs()`가 구현되지 않은 타입을 할당하면 컴파일 에러 발생(위 코드에서 Vertex의 Abs() 코드를 주석처리했을 때

  ```
  cannot use &v (type *Vertex) as type Abser in assignment:
  	*Vertex does not implement Abser (missing Abs method)
  ```

- Interface은 (value, concrete type)의 튜플

### Interface에 nil을 할당하는 경우

- 이때 할당된 value가 nil인 것일뿐, interface 자체는 non-nil(concrete type은 가지고 있다)

```go
package main

import "fmt"

type I interface {
	M()
}

type T struct {
	S string
}

func (t *T) M() {
  if t == nil { // (1) 
		fmt.Println("(nil)")
		return
	}
	fmt.Println(t.S)
}

func main() {
	var i I
	var t *T
	i = t
	fmt.Printf("(%v, %T)\n", i, i) // (<nil>, *main.T) 할당된 value는 nil, concrete type은 non-nil
	i.M() /* 결과 : (nil). 여기서 Null pointer exception 발생하지 않음
 					따라서 메소드 내에서 null pointer에 대한 handler를 작성하는게 일반적 like (1) */
}
```

### Interface 자체가 nil인 경우

- value와 concrete type 모두 nil
- 메소드 호출시 runtime error 발생

```go
package main

import "fmt"

type I interface {
	M()
}

func main() {
	var i I
  fmt.Printf("(%v, %T)\n", i, i) // (<nil>, <nil>) 둘다 nil
  i.M() // runtime error: invalid memory address or nil pointer dereference
}
```

### Empty interface

- 모든 타입의 조상(모든 타입은 0개 이상의 메소드를 가지고 있다.)
- java의 Object class와 유사
- Unknown type을 핸들링할 때 사용 ex) `fmt.Print`

```go
package main

import "fmt"

func main() {
	var i interface{}
  describe(i) // (<nil>, <nil>)
  i = 42 
  describe(i) // (42, int)
	i = "hello"
  describe(i) // (hello, string)
}

func describe(i interface{}) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

### Type assertion

- Type casting과 유사. 인터페이스를 특정 concrete type으로 변환
- Syntax : `interface.(type)`
- Return 값이 하나일 때
  - Type assertion 가능한 경우 : type assertion 결과 리턴 (1)
  - 불가능한 경우 : panic: interface conversion  발생 (2)
- Return 값이 둘 일때
  - Type assertion 가능한 경우 : (결과, true) 리턴 (3)
  - 불가능한 경우 : (해당 type의 zero value, false) 리턴 (panic 발생 x) (4) 

```go
func main() {
	var i interface{} = "hello"

	s := i.(string)
	fmt.Println(s) // (1) hello
	s, ok := i.(string)
	fmt.Println(s, ok) // (3) hello, true
	f = i.(float64) // (2) 
	f, ok := i.(float64)  
	fmt.Println(f, ok) // (4) 0, false
}
```

- Type switch
  - 복수의 Type assertion을 해주는 switch문

```go
func do(i interface{}) {
 	switch v := i.(type) { // type switch 문을 위해 type이라는 예약어 사용해야함
	case int:
		fmt.Printf("Twice %v is %v\n", v, v*2)
	case string:
		fmt.Printf("%q is %v bytes long\n", v, len(v))
	default:
		fmt.Printf("I don't know about type %T!\n", v)
	}
}
```

### Stringer

```go
type Stringer interface {
    String() string
}
```
- string으로 변환할 수 있는 모든 타입을 담은 인터페이스
- `fmt` 패키지 안에 존재
- java의 Object 클래스가 toString()을 갖고 있는 것과 유사
- Type에 String() 메소드를 구현하여 Stringer 인터페이스를 활용하는 함수에서 사용 가능

```go
type Person struct {
	Name string
	Age  int
}

func (p Person) String() string { // String() 클래스 구현 == implements Stringer
	return fmt.Sprintf("%v (%v years)", p.Name, p.Age)
}

func main() {
	a := Person{"Arthur Dent", 42}
	z := Person{"Zaphod Beeblebrox", 9001}
	fmt.Println(a, z)
}

```

### Error

- built-in interface

  ```go
  type error interface {
      Error() string
  }
  ```

- ```go
  i, err := strconv.Atoi("42") // 두번째 인자로 error를 리턴하는 함수
  if err != nil { // 함수 정상 작동시 err == nil
    fmt.Printf("couldn't convert number: %v\n", err) 
    // print함수에서 error의 값을 가져올 때 err.Error()를 호출(err.Error() 우선순위 > err.String() 우선순위)
    return
  }
  fmt.Println("Converted integer:", i)
  ```

### Readers

- `io` package에 정의되어 있음

- Go standard library에 다양한 concrete implementation 존재

  - files, network connections, compressors, ciphers 등등

- `io.Reader`는 `Read()` 메소드 가지고 있음

  ```go
  func (T) Read(b []byte) (n int, err error)
  // byte slice b에 데이터를 채워넣고 채워넣은 수(n) 리턴. 꽉채웠으면 b의 크기와 동일
  // stream 종료시 err로 io.EOF 리턴
  ```

- ```go
  package main
  
  import (
  	"fmt"
  	"io"
  	"strings"
  )
  
  func main() {
  	r := strings.NewReader("Hello, Reader!")
  
  	b := make([]byte, 8)
  	for {
  		n, err := r.Read(b)
  		fmt.Printf("n = %v err = %v b[:n]= %q\n", n, err, b[:n])
  		if err == io.EOF {
  			break
  		}
  	}
  }
  /* 결과
  n = 8 err = <nil> b[:n]= "Hello, R"
  n = 6 err = <nil> b[:n]= "eader!"
  n = 0 err = EOF b[:n]= ""
  */
  ```

  



## References

- <https://tour.golang.org/methods>



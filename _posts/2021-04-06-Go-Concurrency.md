---
title: Go Concurrency
categories:
  - Go
tags:
  - Go
---

## Goroutine

- Go runtime에 의해 관리되는 경량화 스레드
- 메인 goroutine이 종료되면 프로그램이 종료됨



### Channel

- **서로 다른 goroutine 간 데이터 전달(통신) 수단**
- `make`를 통해 초기화해야 사용 가능 
- Channel 자체가 내부에서 mutex처럼 동작 -> 별도의 lock 관리 필요 x
- Sender는 버퍼에 값을 넣을 때까지 block됨(asleep 상태)
- Receiver는 버퍼에 값이 들어올 때까지 block됨(asleep 상태)
- 버퍼가 꽉찬 것 = Receiver가 값을 받을때까지 대기중인 상태

```go
ch := make(chan int) // 채널 버퍼 크기 1
ch <- v    // 채널 ch에 v를 전송한다.
v := <-ch  // ch로 부터 값을 받는다

chs := make(chan int, 2) // 채널 버퍼 크기 2
```

### Deadlock

- **모든 goroutine이 asleep 상태**

	- `fatal error: all goroutines are asleep - deadlock!` 

- ```go
	package main
	
	import "fmt"
	
	func main() {
		ch := make(chan int, 2)
		ch <- 1
		ch <- 2
		ch <- 3 /* fatal error: all goroutines are asleep - deadlock!
					goroutine 1 [chan send] */
		fmt.Println(<-ch)
		fmt.Println(<-ch)
	}
	```

	
	현재 goroutine은 메인 goroutine 하나이고, 이미 꽉찬 채널에 3을 넣으려고 하면 메인 goroutine이 block되어 asleep 상태가 됨 => 모든 goroutine이 asleep이므로 데드락

- ```go
	package main
	
	import "fmt"
	
	func main() {
		ch := make(chan int, 2)
		ch <- 1
		ch <- 2
		fmt.Println(<-ch)
		fmt.Println(<-ch)
		fmt.Println(<-ch) /* fatal error: all goroutines are asleep - deadlock!
							goroutine 1 [chan receive] */
	}
	```

	위와 마찬가지로 메인 goroutine은 하나이고, 빈 채널에서 값을 가져오려고 하면 메인 goroutine이 채널에 값이 들어올 때까지 asleep 상태가 됨 => 모든 goroutine이 asleep이므로 데드락

### Range and Close

```go
package main

import (
	"fmt"
)

func fibonacci(n int, c chan int) {
	x, y := 0, 1
	for i := 0; i < n; i++ {
		c <- x
		x, y = y, x+y
	}
    close(c) // (1)
}

func main() {
	c := make(chan int, 10)
	go fibonacci(cap(c), c)
	for i := range c {
		fmt.Println(i)
	}
}
```

- close(c) : 더 이상 채널 c에 값을 넣지 않겠다고 하는 것

	- sender가 호출하는게 자연스럽고 그래야한다.
	- receiver가 호출 시 sender가 이미 close된 채널에 값을 넣을 수 있고 close된 채널에 값을 넣을시 `panic` 발생

- range c : 채널 c의 receiver 역할

	- ```go
		v, ok := <- ch
		```

		이때, `ok == ( ch에 값이 존재 || !(ch이 closed) )`
	
		즉, ch에 값이 하나도 없고, ch이 close되었을때만 false
	
	- range c 는 위의 ok값이 false가 될 때까지 계속 값을 가져옴
	
	- ```go
		for v := range c
		for v,ok <-ch; ok // 같은 표현
		```

	- 따라서 range loop을 벗어나려면 반드시 close 해주어야함.
	
		- (1)을 주석처리하면  `fatal error: all goroutines are asleep - deadlock!` 발생(유일한 goroutine인 메인 goroutine이 range문에서 asleep상태가 되기 때문)
		- range를 사용하는 등의 상황이 아니면 파일처럼 꼭 닫아줄 필요는 없음


​	

### Select

- 케이스 하나가 runnable할 때까지 block 됨

```go
package main

import "fmt"

func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() { // 익명함수
		for i := 0; i < 5; i++ {
			fmt.Print(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
// 결과 01123quit
```

- case로 default도 지정 가능



## sync.Mutex

- 공유 자원에 대해 하나의 goroutine만 접근가능하도록

- 해당로직의 앞뒤로 lock & unlock

	- ```go
		var mutex sync.Mutex
		mutex.Lock()
		doSomethingWithSharedResource()
		mutex.Unlock()
		```

- ```go
	package main
	
	import (
		"fmt"
		"sync"
		"time"
	)
	
	// SafeCounter is safe to use concurrently.
	type SafeCounter struct {
		mu sync.Mutex
		v  map[string]int
	}
	
	func (c *SafeCounter) Inc(key string) {
		c.mu.Lock()
		// Lock so only one goroutine at a time can access the map c.v.
		c.v[key]++
		c.mu.Unlock()
	}
	
	func (c *SafeCounter) Value(key string) int {
		c.mu.Lock()
		// Lock so only one goroutine at a time can access the map c.v.
		defer c.mu.Unlock()
		return c.v[key]
	}
	
	func main() {
		c := SafeCounter{v: make(map[string]int)}
		for i := 0; i < 1000; i++ {
			go c.Inc("somekey")
		}
	
		time.Sleep(time.Second)
		fmt.Println(c.Value("somekey"))
	}
	```



## References

- <https://tour.golang.org/concurrency>
- <https://gosudaweb.gitbooks.io/effective-go-in-korean/content/concurrency.html>


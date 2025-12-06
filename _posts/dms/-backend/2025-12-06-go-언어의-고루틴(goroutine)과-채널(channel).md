---
layout: post
title: "[Daily morning study] Go 언어의 고루틴(Goroutine)과 채널(Channel)"
description: >
  #daily morning study
category: 
    - dms
    - -backend
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---
# Go 언어의 고루틴(Goroutine)과 채널(Channel)

Go 언어는 병행 프로그래밍(concurrent programming)을 지원하기 위해 고루틴과 채널이라는 두 가지 중요한 기능을 제공합니다. 이 학습 자료에서는 고루틴과 채널이 무엇인지, 어떻게 사용되는지, 그리고 이들을 활용한 간단한 예제를 살펴보겠습니다.

## 1. 고루틴(Goroutine)

고루틴은 가벼운 스레드로, Go 언어에서 동시에 실행할 수 있는 함수를 말합니다. 고루틴은 일반 함수 선언 앞에 `go` 키워드를 붙여 실행합니다. 고루틴은 몇 kilobytes의 메모리만 사용하며, 수천에서 수만 개의 고루틴을 생성할 수 있습니다.

### 1.1. 고루틴 생성

고루틴을 생성하는 기본적인 예제는 다음과 같습니다.

```go
package main

import (
    "fmt"
    "time"
)

func printMessage(msg string) {
    for i := 0; i < 5; i++ {
        fmt.Println(msg)
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    go printMessage("Hello from Goroutine!")
    printMessage("Hello from main!")
}
```

위 코드에서 `go printMessage("Hello from Goroutine!")` 부분은 고루틴을 생성하여 비동기로 `printMessage` 함수를 실행합니다. 

고루틴과 메인 프로그램의 실행은 독립적으로 이루어지므로, 원래 프로그램 흐름에 따라 메인이 먼저 종료될 수도 있습니다. 그러므로 일반적으로 다음과 같이 메인 프로그램이 종료되지 않도록 기다리는 방법을 사용합니다.

### 1.2. 고루틴 대기

고루틴이 끝날 때까지 기다리기 위해 `sync.WaitGroup`을 사용할 수 있습니다.

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func printMessage(msg string, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 5; i++ {
        fmt.Println(msg)
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go printMessage("Hello from Goroutine!", &wg)

    wg.Add(1)
    go printMessage("Hello from main!", &wg)

    wg.Wait() // 모든 고루틴이 끝날 때까지 대기
}
```

위 코드에서는 `sync.WaitGroup`을 사용하여 고루틴이 종료될 때까지 메인 함수가 대기하도록 합니다.

## 2. 채널(Channel)

채널은 고루틴 간에 데이터를 전달하기 위한 구조체입니다. 채널을 통해 고루틴들은 안전하게 서로의 데이터를 주고받을 수 있습니다.

### 2.1. 채널 생성

채널을 생성하려면 `make` 함수를 사용합니다.

```go
ch := make(chan int) // int 타입의 채널 생성
```

### 2.2. 채널에 데이터 보내기 및 받기

채널에 데이터를 보내려면 `<-` 연산자를 사용하고, 데이터를 받으려면 채널 이름 앞에 `<-`를 붙입니다.

```go
package main

import (
    "fmt"
)

func sendData(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i // 데이터 전송
    }
    close(ch) // 채널 닫기
}

func main() {
    ch := make(chan int)

    go sendData(ch)

    for data := range ch {
        fmt.Println(data) // 데이터 수신
    }
}
```

위 예제에서 `sendData` 함수는 채널에 0부터 4까지의 정수를 전송하고, `main` 함수는 채널로부터 데이터를 수신하여 출력합니다. 채널이 닫힐 때까지 `for range` 루프를 사용합니다.

### 2.3. 버퍼드 채널

Go에서는 버퍼가 있는 채널도 지원합니다. 버퍼가 있는 채널의 용량을 지정하여 데이터 전송과 수신이 비동기적으로 이루어질 수 있도록 합니다.

```go
ch := make(chan int, 2) // 용량이 2인 버퍼드 채널 생성
```

이렇게 생성된 채널에서는 두 개의 데이터를 전송한 후에 수신하는 쪽이 데이터를 받기 전까지 블록되지 않습니다.

## 3. 고루틴과 채널을 이용한 통신 예제

고루틴과 채널을 함께 사용하여 두 개의 고루틴 간에 통신하는 예제를 만들어 보겠습니다.

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, ch chan<- string) {
    time.Sleep(time.Second)
    ch <- fmt.Sprintf("Worker %d done", id)
}

func main() {
    ch := make(chan string)

    for i := 1; i <= 3; i++ {
        go worker(i, ch)
    }

    for i := 1; i <= 3; i++ {
        message := <-ch
        fmt.Println(message)
    }
}
```

위의 예제는 세 개의 고루틴을 생성하여 각각의 작업 결과를 채널을 통해 메인 함수로 전달합니다. 메인 함수는 각 작업이 종료될 때까지 데이터 수신을 대기합니다.

## 결론

Go 언어의 고루틴과 채널 기능은 병행 프로그래밍을 단순하게 만들어 주며, 이를 통해 간단한 데이터 통신을 수행할 수 있습니다. 학습 과정에서 다양한 예제를 실험해 보면서 이해를 심화해 나가길 바랍니다.

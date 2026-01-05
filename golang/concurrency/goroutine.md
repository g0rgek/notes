[Soruce code](https://github.com/golang/go/blob/36bca3166e18db52687a4d91ead3f98ffe6d00b8/src/runtime/runtime2.go#L473)
# Что такое горутина ?
## Легковесный поток
  - Динамически расширяемый стек, растет при необходимости
    - [Начальный размер стека](../memory%20model/Стек%20горутины.md#Начальный%20размер%20стека) - 2КБ
    - [Максимальный размер стека](../memory%20model/Стек%20горутины.md#Максимальный%20размер%20стека) - 265mb / 1gig
    - 1000 горутин - 2MB, 1000 тредов - 8GB
  - Инициализация требует намного меньше инструкций, чем для потоков OS
  - Переключение горутин происходит в [GC](../5.%20GC.md)-safe моменты. Не нужно сохраняться все состояние процессора (регистры, флаги) для возобновления работы, достаточно ProgrammCounter, StackPointer, InstructionPointer.
  - Для переключения горутин не нужно выполнять syscall (поход в Kernel Space), делать Mode Switch и обращаться к планировщику OS. Все происходит в User Space
  - Чем плох syscall?
    - Syscall: 2х mode switch + context switch = 1.2-12 micro sec (простаивание)
    - Goroutine context switch = 100-500ns
    - Экономия времени в 100раз максимум
    - когда поток уходит в Kernel Space, программа теряет контроль над ним и не знает когда вернется, будет простаивать
## Структура языка
 [Сущность языка Go](https://github.com/golang/go/blob/36bca3166e18db52687a4d91ead3f98ffe6d00b8/src/runtime/runtime2.go#L473), представлена [структурой](../types/struct.md). 
  - Управляется [планировщиком Go](GMP%20model.md)
  - Содержит состояние кода, который может исполняться конкуретно
    - Поле для хранения [стека горутины](../memory%20model/Стек%20горутины.md)
    - Поле StackGuard - флаг, который выставляется если горутина выполняется дольше 10ms
    - Поле `inMarkAssist` которое показывает помогает ли она раскрашивать кучу [GC](../5.%20GC.md#Алгоритм).
# Утечки горутин
## 1 variant
Писатель не закрыл [канал](chan.md), который [читается](chan.md#Чтение) в другой горутине. Поэтому читающая горутина никогда не завершится и будет потреблять ресурсы.
Чтобы починить - писатель должен закрыть канал.
```go
package main

import (
	"fmt"
	"log"
	"time"
)

func main() {
	doWork := func(strings <-chan string) {
		go func() {
			for str := range strings { // blocked in endless read
				fmt.Println(str)
			}

			log.Println("doWork exited")
		}()
	}

	strings := make(chan string)
	doWork(strings)
	strings <- "Test"
	// close(strings) // add close to fix leaked goroutine

	time.Sleep(time.Second)
	fmt.Println("Done")
}
```
## 2 variant
Делаем запрос в 5 реплики и возвращаем ответ от первой ответившей. В цикле запускаем 5 горутин, которые пишут в канал. 
Если использовать [небуферезированный канал](chan.md#Unbuffered%20chan%20(synchronous)), то одна из горутин успеет записать в канал и функция завершится. Но четыре оставшиеся заблокируются на запись **НАВСЕГДА** 
Чтобы починить:
1. Добавить каналу [буфер](chan.md#Buffered%20chan%20(asynchronous)). Это допустимо, так как даже если в канале есть значения, но на него никто не ссылается - [GC](5.%20GC.md) его очистит в будущем.
2. ИЛИ использовать [select](select.md) with default case.
```go
package main

// First-response-wins strategy
func request() int {
	ch := make(chan int) // fix1: add buffer
	for i := 0; i < 5; i++ {
		go func() {
			ch <- i // 4 goroutines will be blocked
		}()
	}

	return <-ch
}
```
## 3 variant
```go
// Example program showing a goroutine leak. It launches a
// goroutine that sends on a channel but sometimes there is
// no other goroutine available to receive.
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"runtime"
	"time"
)

func main() {

	// Capture starting number of goroutines.
	startingGs := runtime.NumGoroutine()

	if err := process("gophers"); err != nil {
		log.Print(err)
	}

	// Hold the program from terminating for 1 second to see
	// if any goroutines created by process terminate.
	time.Sleep(time.Second)

	// Capture ending number of goroutines.
	endingGs := runtime.NumGoroutine()

	// Report the results.
	fmt.Println("========================================")
	fmt.Println("Number of goroutines before:", startingGs)
	fmt.Println("Number of goroutines after :", endingGs)
	fmt.Println("Number of goroutines leaked:", endingGs-startingGs)
}

// result wraps the return values from search. It allows us
// to pass both values across a single channel.
type result struct {
	record string
	err    error
}

// process is the work for the program. It finds a record
// then prints it. It fails if it takes more than 100ms.
func process(term string) error {

	// Create a context that will be canceled in 100ms.
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	// Make a channel for the goroutine to report its result.
	ch := make(chan result)

	// Launch a goroutine to find the record. Create a result
	// from the returned values to send through the channel.
	go func() {
		record, err := search(term)
		ch <- result{record, err} // горутина заблокируется тут
	}()

	// Block waiting to either receive from the goroutine's
	// channel or for the context to be canceled.
	select {
	case <-ctx.Done():
		return errors.New("search canceled") //Если контекст завершится раньше, чем выполнение горутины - мы выйдем из функции и горутина повиснет навсегда
	case result := <-ch:
		if result.err != nil {
			return result.err
		}
		fmt.Println("Received:", result.record)
		return nil
	}
}

// search simulates a function that finds a record based
// on a search term. It takes 200ms to perform this work.
func search(term string) (string, error) {
	time.Sleep(200 * time.Millisecond)
	return "some value", nil
}

```
### Fixed
```go
// Example program showing a goroutine leak. It launches a
// goroutine that sends on a channel but sometimes there is
// no other goroutine available to receive.
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"runtime"
	"time"
)

func main() {

	// Capture starting number of goroutines.
	startingGs := runtime.NumGoroutine()

	if err := process("gophers"); err != nil {
		log.Print(err)
	}

	// Hold the program from terminating for 1 second to see
	// if any goroutines created by process terminate.
	time.Sleep(time.Second)

	// Capture ending number of goroutines.
	endingGs := runtime.NumGoroutine()

	// Report the results.
	fmt.Println("========================================")
	fmt.Println("Number of goroutines before:", startingGs)
	fmt.Println("Number of goroutines after :", endingGs)
	fmt.Println("Number of goroutines leaked:", endingGs-startingGs)
}

// result wraps the return values from search. It allows us
// to pass both values across a single channel.
type result struct {
	record string
	err    error
}

// process is the work for the program. It finds a record
// then prints it. It fails if it takes more than 100ms.
func process(term string) error {

	// Create a context that will be canceled in 100ms.
	ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
	defer cancel()

	// Make a channel for the goroutine to report its result.
	ch := make(chan result)

	// Launch a goroutine to find the record. Create a result
	// from the returned values to send through the channel.
	go func() {
		record, err := search(ctx, term)
		ch <- result{record, err}
	}()
	res, _ := <-ch
	return res.err
}

// search simulates a function that finds a record based
// on a search term. It takes 200ms to perform this work.
func search(ctx context.Context, term string) (string, error) {
	// go func(){
	// case <- ctx.Done():
	// 	return "", errors.New("search canceled")
	// }()

	go func() (string, error) {
		select {
		case <-ctx.Done():
			return "", errors.New("search canceled")

		}
	}()

	time.Sleep(200 * time.Millisecond)

	return "some value", nil
}

```
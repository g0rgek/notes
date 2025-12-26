# Свойства
- Используется для чтения значений из нескольких [[chan]] сразу.
- Если значения одновременно пришли в несколько case - он выбирается рандомно.
- select **не выбирает** case, которые ведут к **блокировке**. Если есть ветка default - она всегда будет отрабатывать, если другие ведут к блокировке. Это достигается устройством внутренней функции `chansend`/`chanrecv` у [chan (source)](https://go.dev/src/runtime/chan.go) . Они возвращают флаги будет ли действие блокирующим/неблокирующим без блокировки mutex у канала (fast path).
# Устройство case
[Source code](https://go.dev/src/runtime/select.go)
```go
type scase struct {
	c *hchan
	elem unsafe.Pointer
}
```
# Select with break and continue
Внутри `for select` конструкции:
- **break** - выходит из select
- **continue** - переходит на след. итерацию for
```go
package main

import "fmt"

func main() {
	data := make(chan int)
	go func() {
		for i := 1; i <= 4; i++ {
			data <- i
		}
		close(data)
	}()

	for {
		value := 0
		opened := true
		
		select {
		case value, opened = <-data:
			if value == 2 {
				continue
			} else if value == 3 {
				break
			}
			
			if !opened {
				return
			}
		}
		
		fmt.Println(value)
	}
}
```
# Prioritization
Если необходимо приоритизировать чтение из одного из каналов, можно использовать паттерны:
1. **Distinct select**: отдельный select для избранного канала с default. В цикле, если есть значение в `ch1` - обрабатываем и выходим. Иначе проваливаемся в основной select и ждем значения из `ch1` и `ch2`
2. **Weighted chan**: несколько case с тем же каналом. Чем больше case - тем больше приоритет.
```go
package main

import (
	"fmt"
	"time"
)

func producer(ch chan<- int) {
	for {
		ch <- 1
		time.Sleep(time.Second)
	}
}

func main() {
	ch1 := make(chan int) // more prioritized
	ch2 := make(chan int)

	go producer(ch1)
	go producer(ch2)

	for {
		select {              // distinct select
		case value := <-ch1:
			fmt.Println(value)
			return
		default:
		}
		
		select {
		case value := <-ch1:   // weighted case
			fmt.Println(value)
			return
		case value := <-ch1:
			fmt.Println(value)
			return
		case value := <-ch2:
			fmt.Println(value)
		}
	}
}
```
# TryWrite & TryReceive
Если чтение или запись не является блокирующей операцией для этого канала (есть данные/не заполнен), то произведется действие и блокировки не будет. Иначе будет исполнен код default ветки и вернется флаг. Потому что select умный (читать свойства).
```go
package main

func tryToReadFromChannel(ch chan string) (string, bool) {
	select {
	case value := <-ch:
		return value, true
	default:
		return "", false
	}
}

func tryToWriteToChannel(ch chan string, value string) bool {
	select {
	case ch <- value:
		return true
	default:
		return false
	}
}

// Можно читать и писать в рамках одного select
func tryToReadOrWrite(ch1 chan string, ch2 chan string) {
	select {
	case <-ch1:
	case ch2 <- "test":
	default:
	}
}
```
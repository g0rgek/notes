-  примитив синхронизации для ожидания услови(я|ий) без [busy-waiting](https://en.wikipedia.org/wiki/Busy_waiting) с возможностью единичного и множественного пробуждений
- Блокирует горутину(ны) до момента поступления сигнала от другой горутины о выполнении некоторого условия.
- Разница с [[chan]]:
	Broadcast-семантика: позволяет несколько раз оповещать N горутин. А [метод close канала](<chan.md#Закрытие>)  позволяет сделать это только единожды. Но если делать с отправками данных, то  это O(n) отправок, и нет гарантии, что проснутся именно все — одна быстрая горутина может выхватить несколько сигналов подряд. 
# API
- Оповещение всех: `Broadcast()`
- Оповещение одной рандомной по принципу FIFO: `Signal()`
- Ожидание сигнала: `Wait()`
	- Если вызвать `Wait()` и не отправлять сигнал - **deadlock**
	- Горутина отпускает мьютекс, встаёт в очередь ожидания и паркуется. Распарковывается только когда кто-то вызовет Signal() или Broadcast() [Source code](https://go.dev/src/runtime/sema.go#L581)
# Преимущества
- Ожидание сложного условия: поток должен проснуться только когда: буфер не пуст, конфиг перезагружен и количество активных воркеров не превышает лимит
- Возможность FIFO-порядка пробуждения — `Signal()` будит горутины строго в порядке засыпания
# Example
Subscriber подписывается на события (появления записей в [[map]]).

Важно понимать, что в функции `subscribe()`, при вызове `c.Wait()` [[mutex]] разблокируется "под капотом". Пока горутина спит, нет смысла в захвате мьютекста. Когда в `publish()` вызовется `Broadcast()` и разбудятся горутины - они снова захватят мьютекс и продолжат выполнение.
```go
package main

import (
	"log"
	"sync"
	"time"
)

func subscribe(name string, data map[string]string, c *sync.Cond) {
	c.L.Lock()
	for len(data) == 0 { // spinlock
		c.Wait()         // goroutine sleeps, mutex Unlocks
	}
	log.Printf("[%s] %s\n", name, data["key"])
	c.L.Unlock()
}

func publish(name string, data map[string]string, c *sync.Cond) {
	time.Sleep(time.Second)
	
	c.L.Lock()
	data["key"] = "value"
	c.L.Unlock()
	
	log.Printf("[%s] data publisher\n", name)
	c.Broadcast() // wake up all waiting goroutines
}

func main() {
	data := map[string]string{}
	cond := sync.NewCond(&sync.Mutex{})

	wg := sync.WaitGroup{}
	wg.Add(3)

	go func() {
		defer wg.Done()
		subscribe("subscriber_1", data, cond)
	}()

	go func() {
		defer wg.Done()
		subscribe("subscriber_2", data, cond)
	}()

	go func() {
		defer wg.Done()
		publish("publisher", data, cond)
	}()

	wg.Wait()
}
```
# Wait method code
```go
func (c *sync.Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()                         // unlocks mutex
	runtime_notifyListWait(&c.notify, t) // blocks here until broadcast
	c.L.Lock()
}
```
# Semaphore 
Аналогичная логика [[chan#Simple semaphore]]

```go
package main

import "sync"

type Semaphore struct {
	count int
	max int
	cond *sync.Cond
}

func NewSemaphore(limit int) *Semaphore {
	return &Semaphore{
		max: limit,
		cond: sync.NewCond(&sync.Mutex{})
	}
}

func (s *Semaphore) Acquire() {
	s.cond.L.Lock()
	defer s.cond.L.Unlock()
	
	for s.count >= s.max {
		s.cond.Wait()
	}
	
	s.count++
}

func (s *Semaphore) Release() {
	s.cond.L.Lock()
	defer s.cond.L.Unlock()
	
	s.count--
	s.cond.Signal()
}
```
# Cond игнорирует [[context|context]]
Cond появился в [доисторические времена](https://go.googlesource.com/go/+/05b1dbd0a6994f6ba9ab3505fce1abff93606d9e#:~:text=commit%2005b1dbd0a6994f6ba9ab3505fce1abff93606d9e,diff), до появление пакета [context](https://go.dev/doc/go1.7#context) и вплоть до версии Go 1.21 не было поддержки реагирования на context.Context внутри метода Wait:
```go
func (q *Queue) Get() any {
    q.cond.L.Lock()
    defer q.cond.L.Unlock()
    
    for len(q.items) == 0 {
        q.cond.Wait()
    }
    
    return q.pop()
}

// Вызывающий код
func handler(ctx context.Context) error {
    item := queue.Get()  // Никак не реагируем на контекст
    
    // Если очередь пуста — висим вечно,
    return process(item)
}
```
Чем это грозит:
— Горутины утекают и копятся до OOM Killed
— Graceful shutdown не работает — процесс убивается по таймауту
— Клиент ушёл, а сервер продолжает держать ресурсы

В качестве решения можно использовать Watcher-горутину:
```go
func (q *Queue) Get(ctx context.Context) (any, error) {
    done := make(chan struct{})
    // Закрываем канал чтобы убить Watcher-горутину    
    defer close(done)

    // Watcher: при отмене контекста распарковываем
    // все ожидающие потоки
    go func() {
        select {
        case <-ctx.Done():
    // Гарантируем, что поток(и) запаркован(ы) (отдал мьютекс)
            q.cond.L.Lock()
            q.cond.Broadcast()
            q.cond.L.Unlock()
        case <-done:
        }
    }()
  
    q.cond.L.Lock()
    defer q.cond.L.Unlock()
    
    for len(q.items) == 0 {
        q.cond.Wait()
        // Проверяем контекст после сигнала
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
        }
    }
    
    return q.pop(), nil
}
```
В версии Go 1.21 в пакете context появилась альтернатива в виде [context.AfterFunc](https://pkg.go.dev/context#example-AfterFunc-Cond).
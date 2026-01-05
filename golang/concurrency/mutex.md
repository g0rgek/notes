# API
- Создание: `var mu sync.Mutex`
- Блокировка: `mu.Lock()`
- Разблокирование: `mu.Unlock()`

- Если [горутина](goroutine.md) захватит **УЖЕ** захваченный mutex, то она поменяет свое состояние с `Running -> Waiting`.
- Если разблокировать разблокированный мьютекс - **fatal error**

# Lock/Unlock order
Для каждой переменной `l` типа `sync.Mutex` или `sync.RWMutex` каждый вызов `l.Unlock()` синхронизирован и вернется до того, как вернется `l.Lock()`.

Пример гарантированно выведет «hello, world». Первый вызов `l.Unlock()` синхронизируется до того, как завершится второй вызов `l.Lock()`, который выполняется до функции `print(a)`.
Т.е. гортина заблокируется при втором вызове `l.Lock()` и разблокируется после вызова `l.Unlock()`:
```go
var l sync.Mutex
var a string

func f() {
 a = "hello, world"
 l.Unlock()
}

func main() {
 l.Lock()
 go f()
 l.Lock()
 print(a)
}
```
## Пример [FooBarAlternately](https://leetcode.com/problems/print-foobar-alternately/)
Задача: распечатать Foo и Bar последовательно, друг за другом, из разных потоков.

Для этого мы создаем 2 мьютекса, и лочим `barMu` перед возвратом объекта `FooBar`. Тогда, если первым будет вызван метод `Bar`, то горутина заблокируется на вызове `fb.BarMu.Lock()`.  Первым исполнится метод `Foo`, распечатает "foo" и в конце разблокирует `barMu`. Таким образом, получим последовательный вывод "foo" и "bar".
```go
type FooBar struct {
	number int
	fooMu  sync.Mutex
	barMu  sync.Mutex
}

func NewFooBar(number int) *FooBar {
	fb := &FooBar{number: number}
	fb.barMu.Lock()
	return fb
}

func (fb *FooBar) Foo(printFoo func()) {
	for range fb.number {
		fb.fooMu.Lock() // will block until Unlock is called
		printFoo()
		fb.barMu.Unlock()
	}
}

func (fb *FooBar) Bar(printFoo func()) {
	for range fb.number {
		fb.barMu.Lock() // will block until Unlock is called
		printFoo()
		fb.fooMu.Unlock()
	}
}
```

# WaitQueue
Внутри mutex используется очередь заблокированных горутин. Она представлена как сбалансированное дерево. Реализована [структурой sudog](chan.md#sendq%20&%20recvq), которая используется в waitq/recvq у каналов.
```go
// Asynchronous semaphore for sync.Mutex.

// A semaRoot holds a balanced tree of sudog with distinct addresses (s.elem).
// The operations on the inner lists of sudogs with the same address
// are all O(1). The scanning of the top-level semaRoot list is O(log n),
type semaRoot struct {
	lock  mutex
	treap *sudog        // root of balanced tree of unique waiters.
	nwait atomic.Uint32 // Number of waiters. Read w/o the lock.
}
```
# Mutex modes
[Article](https://victoriametrics.com/blog/go-sync-mutex/)
Мьютекс может пребывать в нескольких состояниях:
```go
const (
	mutexLocked = 1 << iota // mutex is locked
	mutexWoken
	mutexStarving
	mutexWaiterShift = iota
)
starvationThresholdNs = 1e6
```
## Normal mode
- Лучше для производительности, потому что одна горутина может захватить мьютекс даже при ожидающих в очереди
- Ожидающие горутины стоят в очереди FIFO, но пробуждённый ожидающий не получает мьютекс автоматически.
- Новые горутины могут опередить его, так как уже выполняются на CPU.
- Если горутина ожидает более ~1 мс, мьютекс переключается в режим голодания.
## Starvation mode
- Разблокирующая мьютекс горутина передаёт владение напрямую следующему ожидающему.
- Новые горутины **не пытаются** захватить мьютекс и просто становятся в конец очереди.
- Режим голодания продолжается до тех пор, пока очередь не уменьшится на достаточное количество, затем мьютекс переходит в нормальный режим.

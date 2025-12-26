# Определение
Это фрагмент кода, к которому можно обратиться из другого места программы. С именем функции связан первый адрес инструкции, с которой начинается функция.

- Порядок вычисления аргументов - слева направо.
- Функция является first-class citizen (функция первого порядка) - 
	- можно: присваивать, передавать и возвращать;
	- можно хранить в куче и обращаться как к переменной!!!
	```go
	func answer() int{return 42}
	func main() {
		a := answer
		println(a())
	}
	```
	- нельзя: сравнивать, брать адрес;
- [[zero value]] функции - [[nil]]
- Есть прототипы функций (объявление без тела). Реализацию можно написать на ASM.
- Нельзя перегружать функции
- Нет параметров по умолчанию
- Нет RVO(return value optimization) и NRVO (named return optimization).  Если функция возвращает инициализированный объект, мы копируем это значение при возврате.  Можно передать в функцию указатель на этот объект и инициализировать (copy elision).
- Нет хвостовой рекурсии. Когда функция заканчивается вызовом самой себя, тогда компилятор разворачивает в цикл с одним стек-фреймом, просто меняя локальные переменные от итерации к итерации.
# Передача аргументов
Все аргументы при [передаче в функцию](<Passing by value.md>) **копируются**!
# Variadic parameters
Функция может принимать аргументы переменной длины последними в списке аргументов. На самом деле, это [[slice]] аргументов.
```go
func process(data string, tokens ...int){
	// impl...
}

func main() {
	process("")
	process("", 1)
	process("", 1, 2)
	process("", []int{1,2,3}...)
}
```

Даже если передать nil, то ничего не сломается:
```go
package main

import "fmt"

func process(values ...*int){
	fmt.Println(len(values))
}

func main() {
	process(nil)    // 1
	process(nil...) // 0
}
```
# Именованные возвращаемые результаты
По факту, `x` и `y` алоцированы на стеке функции. К ним можно обращаться как к обычным переменным, это синтаксический сахар.
```go
func getCoord() (x,y int){
	x=5
	y=6
	return
}
```
# Встраивание функций
Если функция достаточно мала (дешева), то компилятор может встроить ее инструкции в место вызова. Тогда, не будет никакого вызова функции и накладных расходов на это.

Есть бюджет, равный 80, на встраивание одной функции. Считаем стоимость функции - сумма всех инструкций, помноженная на какой-то вес. Если стоимость меньше или равна бюджету, мы можем встроить функцию.

Директива `//go:noinline` запрещает делать встраивание этой функции компилятору.
```go
package main

import "testing"

// go test -bench=. comparison_test.go

//go:noinline
func maxWithoutInline(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func maxWithInline(a, b int) int {
	if a > b {
		return a
	}
	return b
}

func BenchmarkMaxWithoutInline(b *testing.B) {
	var r int
	for i := 0; i < b.N; i++ {
		r = maxWithoutInline(-1, i) // 0.8050 ns/op
	}
	_ = r
}

func BenchmarkMaxWithInline(b *testing.B) {
	var r int
	for i := 0; i < b.N; i++ {
		r = maxWithInline(-1, i)    // 0.2335 ns/op
	}
	_ = r
}
```
## [[defer]] prevents inline
```go
func defered(a, b int) int { // cannotInlineFunction (op DEFER)
	defer fmt.Println("jopa")
	if a > b {
		return a
	}
	return b
}
```
### Slow path outline
Прием называется Slow Path outline. Когда мы выносим сложную логику в другую функцию, а перед ее вызовом, в маленькой функции, проверяем оптимистичные случаи.
defer **не позволит** заинлайнить эту оптимистичную проверку:
```go
package main

import "sync"

type S struct{
	mu sync.Mutex
	value int
}

func (s *S) ModifyFast(val int){
	s.mu.Lock()
	defer s.mu.Unlock()
	if val == 69{
		return
	}
	s.modifySlow(val)
}

func (s *S) modifySlow(val int){
	// 100+ loc
}

// go build -gcflags="-m=2"
func main(){
	s := S{}
	s.ModifyFast(69) //cannot inline (*S).
	
}
```

Такой подход используется в [[mutex]]:
```go
func (m *Mutex) Lock() {
	// Fast path: grab unlocked mutex.
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path (outlined so that the fast path can be inlined)
	m.lockSlow()
}
```
## [[panic и recover#recover]] prevents inline

```go
func recovered(a, b, weighth int) int { // cannotInlineFunction (call to recover)
	if r := recover(); r != nil {
	}
	if a > b {
		return a
	}
	return b
}
```
## go prevents inline
```go
func (t *tickerProcesser) run() { // cannotInlineFunction (op GO)
	go func() {
		for range t.ticker.C {
			t.registry.Send(t.tName, t.target, t.msg)
		}
	}()
}
```
# Анонимные функции
[[Замыкание захватывает по указателю]]
# Функции высшего порядка
Это функция, которая принимает другие функции как входные аргументы или возвращает функцию.
```go
func forEach(data []int, fn func(int) int) []int {
	newData := make([]int, 0, len(data))
	for _, value := range data {
		newData = append(newData, fn(value))
	}
	
	return newData
}
```
# Читая функция
Функция, которая является детерминированной и не обладает побочными эффектами. 
```go
func pureFunc(x,y int) {
	return x + y
}
```

Функция **не является** чистой, если она:
- изменяет входные значения (разыменование указателей)
- логирует
- использует глобальные переменные
- зависит от переменных окружения
- использует рандомизацию
# Паттерны функционального программирования
## Генератор
### Последовательный
Функция, которая ссылается на свободные переменные области видимости своей родительской функции. `Generator` является функцией высшего порядка, она возвращает  анонимную функция (замыкание). 
И это [[Замыкание захватывает по указателю]] переменную number. Очевидно, произойдет [[Escape analysis#Случаи когда 100% будет escape]]. Потому что мы вышли из области видимости `func Generator()`, но продолжаем ссылаться на `number`. 
В итоге, будем получать последовательные значения при каждом вызове функции.
```go
package main

import "fmt"

func Generator(number int) func() int { // moved to heap: number
	return func() int {                 // func literal escapes to heap
		r := number
		number++
		return r
	}
}

func main() {
	generator := Generator(100)
	for i := 0; i <= 200; i++ {
		fmt.Println(generator())
	}
}
```
### Псевдорандомный
```go
package main

import (
	"fmt"
)

func produce(source int, permutation func(int) int) func() int { //moved to heap: source, leaking function permutation
	return func() int { // func literal escapes to heap
		source = permutation(source)
		return source
	}
}

func mutate(number int) int {
	return (1664525*number + 1013904223) % 2147483647
}

func main() {
	next := produce(1, mutate)

	fmt.Println(next())
	fmt.Println(next())
	fmt.Println(next())
	fmt.Println(next())
	fmt.Println(next())
}
```
## Ленивые вычисления
Создание [[map]] будет произведено только при первом вызове `data()`:
```go
package main

import (
	"fmt"
)

type LazyMap func() map[string]string

func Make(ctr func() map[string]string) LazyMap {
	var initialized bool
	var data map[string]string
	return func() map[string]string {
		if !initialized {
			data = ctr()
			initialized = true
			ctr = nil // for GC
		}

		return data
	}
}

func main() {
	data := Make(func() map[string]string {
		return make(map[string]string)
	})
	
	data()["key"] = "value"
	fmt.Println(data())
}
```
## Карирование
Можно создать логгер и первым аргументов передавать уровень логгирования. А вторым - сообщение.
```go
package main

import "fmt"

func log(severity string) func(msg string) {
	return func(msg string) {
		fmt.Printf("[%s]: %s", severity, msg)
	}
}

func main() {
	logInfo := log("INFO")
	logInfo("all working!") //[INFO]: all working!
}
```
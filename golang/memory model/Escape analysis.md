# Определение
Процесс во время компиляции, который определяет где в программе создаются объекты и как они используются

Escape Analysis пытается оставлять всё на [стеке горутины](../memory%20model/Стек%20горутины.md) Но в некоторых случаях, переменные убегают в кучу. Строится взвешенный граф, относительно используемых объектов:
- вершины - переменные
- взвешенные ребра - сумма по операциям `&` и `*`
Если сумма не нулевая - переменная убегает в кучу.
# Размеры переменных на стеке
Максимальный размер переменной:
- VarSize (`var x T; x:=`) - 10МБ
- ImplicitVarSize (`new(T); &T(); make([]T,); []byte{...}`) - 64КБ
# Случаи когда будет escape
- Передача переменной в `fmt.Println()` (использует reflection)
- Возвращаем указатель из функции (prevent dangling pointer)
- Возвращаем из функции структуру с указателем ([slice](../types/slice.md))
- Если одно из полей структуры должно быть в куче- то вся структура улетит в кучу
- Если один из элементов [array](../types/array.md) / [slice](../types/slice.md) улетит в кучу - то все элементы тоже
- Функция [HeapAllocationReason](https://github.com/golang/go/blob/master/src/cmd/compile/internal/escape/utils.go#L173)
# Проверить escape объектов
Чтобы проверить какое-либо предположение по решению компилятора, используйте флаг:
- `-m=number` = print optimization decisions, number changes verbosity
- `-l` = disable inlining
```sh
go build -gcflags '-m=1 -l' -o app .
```
# Примеры
## io.Reader
[Understanding allocations - GopherCon SG 2019](https://www.youtube.com/watch?v=ZMZpH4yT7M0)

Метод `Read()` [интерфейса](../types/interface.md) `io.Reader` устроен так:
```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```
Он принимает слайс байт и возвращает кол-во прочитанных байт или ошибку.

Если мы будем возвращать `[]byte` из функции, то он, скорее всего, убежит в кучу:
```go
package main

func read() []byte{
	b := make([]byte, 32) //./main.go:4:11: make([]byte, 32) escapes to heap
	return b
}

func main() {
	b := read()
	_ = b
}
```

Чтобы этого не было, лучше передавать `[]byte` в функцию и записывать в него. Такой подход позволит хранить [slice](slice.md) на стеке.
```go
package main

func read(b []byte) int{ //./main.go:3:11: b does not escape
	// Implementation
	return 0
}

func main() {
	b := make([]byte, 32) //./main.go:9:11: make([]byte, 32) does not escape
	_ = read(b)
}
```
## Indirect assignment
[Разбор](https://www.ardanlabs.com/blog/2018/01/escape-analysis-flaws.html#:~:text=the%201.9%20compiler.-,Indirect%20Assignment,-The%20%E2%80%9CIndirection%20Assignment)
Показательный пример работы Escape Analysis. Идентичный код с разницей в одну строку.
Но в первом случае number остается на стеке, а во втором - убегает в кучу.
```go
package main

import "testing"

// go test -bench=. -benchmem

type Data struct {
	pointer *int
}

//BenchmarkIteration1  1000000000    0.2325 ns/op    0 B/op       0 allocs/op
func BenchmarkIteration1(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var number int
		data := &Data{
			pointer: &number,
		}
		_ = data
	}
}

//BenchmarkIteration2  217878158      5.385 ns/op    8 B/op       1 allocs/op
func BenchmarkIteration2(b *testing.B) {
	for i := 0; i < b.N; i++ {
		var number int
		data := &Data{}
		data.pointer = &number
		_ = data
	}
}
```

В отчете escape-analysis, причина по которой `number` убегает в кучу называется `(star-dot-equals)`. Предполагаю, что это необходимость компилятору выполнить такую операцию:
```go
(*data).pointer = &number
```
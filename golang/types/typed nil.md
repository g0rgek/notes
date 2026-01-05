У переменной есть ТИП и ЗНАЧЕНИЕ
```go
var NAME TYPE = VALUE
```

Инициализация без значения:
```go
var num int // TYPE != nil, VALUE != nil
```

Инициализация без типа:
```go
var num = 12 // TYPE != nil, VALUE != nil
```

Инициализация интерфейсным типом:
```go
var e1 error // error - это ИНТЕРФЕЙС, а не реальный тип
			 // Type = nil, Value = nil
```


```go
package main

import (
	"fmt"
)

type errorString struct {
	s string
}

func (e errorString) Error() string {
	return e.s
}

func checkErr(err error) {
	fmt.Println(err == nil) // true if value = type = nil
}

func main() {
	var e1 error // Type = nil, Value = nil
	checkErr(e1) // true

	var e *errorString // Type = *errorString, Value = nil
	checkErr(e) // false

	e = &errorString{} // Type = *errorString, Value = ""
	checkErr(e) // false

	e = nil // Type = *errorString, Value = nil
	checkErr(e) // false
}

```

```
Опасная штука. Сделать ресивер указателем (e *myError) Error() и if (err != nil) fmt.Println(err.Error()) приведет к панике. Поэтому если ошибок нет, то явно возвращать nil. Я так исходящий тип у функции поменял: func bar() error поменял на bar() *myError. А чо, компилируется же и даже внутренность bar() не поменялась. Как было внутри bar() выражение return nil так и осталось, указатель же. Вот только смысл вызова foo(bar()) где foo(e error) начисто поменялся. Компилер присоседил тип к транзитному nil и foo сломалось. В гохе nil как указатель и nil для интерфейсов разные вещи.
```

```go
package main

import "fmt"

type myError struct {
	message string
}

func (e *myError) Error() string {
	return e.message
}

func bar() *myError {
	return nil 
}

func foo(e error) {
	if e != nil {
		fmt.Println(e.Error()) // Calling method from nil 
	} else {
		fmt.Println("No error")
	}
}

func main() {
	err := bar() // Type: *myError, Value: nil
	foo(err) // PANIC!!!
}

```
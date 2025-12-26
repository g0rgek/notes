# При передаче по указателю в функцию в Golang - указатель копируется!!! Те создается локальная переменная, которая содержит адрес на изначальную переменную. Чтобы модифицировать изначальную переменную, нужно разименовать этот указатель и положить новое значение.
# Number
```go
package main

import (
	"fmt"
)

func main() {
	v := 5
	
	p := &v
	fmt.Println(*p)

	changePointer(p)
	fmt.Println(*p)
}

func changePointer(p *int) {
	v := 3
	p = &v
}

```

В функции main имеем кадр стека: 
    main
|------------------|
| v = 5                     |
| p = &v = 0xff212 |
|------------------|
При вызове функции changePointer, создается отдельный кадр стека:
    main
|------------------|
| v = 5                     |
| p = &v = 0xff212 |
|------------------|
   changePointer
|--------------------------------|
| p=0xff212   // local var                |
| v = 3            // local var                |
| p = &v = 0xffaaa                          |
|--------------------------------|
После возвращения из функции, локальные переменные p, v инвалидируются.

Чтобы исправить работу функции, нужно разименовать указатель p и положить зна
чение локальной переменной v.
## Fix 1
```go
func changePointer(p *int) {
	v := 3
	*p = v
}
```

# Struct
```go
package main

import "fmt"

type Person struct {
	Name string
}

func changeName(person *Person) {
	person = Person{
		Name: "Olga",
	}
}

func main() {
	person := &Person{
		Name: "Eugene",
	}
	fmt.Println(person.Name)
	changeName(person)
	fmt.Println(person.Name)
}

```

## Fix 1
```go
package main

import "fmt"

type Person struct {
	Name string
}

func changeName(person *Person) {
	*person = Person{
		Name: "Olga",
	}
}

func main() {
	person := &Person{
		Name: "Eugene",
	}
	fmt.Println(person.Name)
	changeName(person)
	fmt.Println(person.Name)
}
```
## Fix 2
```go
package main

import "fmt"

type Person struct {
	Name string
}

func changeName(person *Person) {
	person.Name = "Olga"
}

func main() {
	person := &Person{
		Name: "Eugene",
	}
	fmt.Println(person.Name)
	changeName(person)
	fmt.Println(person.Name)
}

```
# Array/Slice
При передаче массива в функцию - содержимое копируется. Если передавать большие массивы, то это приведет к большому расходу памяти и работе GC. 
Лучше передавать указатель на [[array]] и как [[slice]].
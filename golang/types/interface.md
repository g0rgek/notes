# Header
```go
// 2 machine words, 16 byte
type iface struct{
	tab *itab           // Информация об интерфейсе
	data unsafe.Pointer // Хранимые данные (реализатор интерфеса)
}

// itab хранит тип интерфейса и информацию о хранимом типе
// уникальна для каждой пары interface -> static type
type itab struct{
	inter *interfacetype // Метаданные интерфейса
	_type *_type         // Gо-шный тип хранимого интрефейсом значения
	hash uint32
	_[4]byte            // padding, выравнивание поля hash до 8 байт
	fun [1]uintptr      // Список методов типа, удовлетворяющих интерфейсу
}
```
Интерфейс - это контракт, абстракция над базовым типов. Используются для уменьшения связанности кода(decoupling), для упрощения тестирования. 
Заменяют [[Полиморфизм]] из [[OOP]].
Про [padding](<struct.md#Padding>) и зачем он нужен для поля hash.
```
type User interface {
	Name() string
	Age() int
}

p := Person{"John", 25}
user := User(p)
┌─────────────┬─────────────┐                 
│             │             │                 
│   0xA00B0   │   0xA00B8   │                 
│    (itab)   │   (data)    │                 
│             │             │                 
└──────┬──────┴───────────┬─┘                 
       │                  │                   
       │                  │                   
 itab  │           Person │                   
┌──────▼──────┐   ┌───────▼────┐   ┌──────────┐
│             │   │  0x0BCC44  │   │          │
│ iface table │   │ (str data) ├───►  "John"  │
│  structure  │   │            │   │          │
│             │   ├────────────┤   └──────────┘
└─────────────┘   │     4      │                            
                  │ (str len)  │                          
                  ├────────────┤                      
                  │     25     │                            
                  └────────────┘               
```

Массив длинной в единицу там это реализация variable length array. Нужно это чтобы элементы располагались в структуре последовательно. Если использовать вместо этого слайс то при итерации элементов придется постоянно прыгать по указателю, хранящемуся в слайсе, а потом еще по указателю на сам метод. С оптимизацией в виде variable length array мы добиваемся того, что убираем лишний прыжок по указателю (если бы использовали слайс или просто указатель) и позволяем первым 5 методам поместиться в кеш линию (64 байта на большинстве современных процессорах), 5 т-к первые 24 байта структуры itab заняты метаинформацией типов и хешем с паддингом, а один указатель занимает 8 байт на 64 битных системах. 
# Late binding
Для каждой парты interface->static type, табличка`*itab` будет уникальной. Чтобы не просчитывать это всё на этапе компиляции, компилятор генерирует метаданные для каждого статического типа, в которых хранится список методов типа.
Аналогично, генерируются метаданные со списком методов для каждого интерфейса. Во время исполнения программы, Go может вычислить itable на лету для каждой пары, затем кешируется.
**Максимальное** количество методов у интерфейса **65536**
# Constraints
До `go v1.18` все типы интерфейсов можно было использовать как типы значений, но потом некоторые типы интерфейсов можно использовать только как ограничения типов (constraints):
```go
type AnyByteSlice interface {
	~[]byte
}

type Unsigned interface {
	uint | uintptr | uint8 | uint16 | uint32 | uint64
}
```
# Duck typing
В отличие от Rust, в Go нет отдельного `impl` блока, в котором описываются все [методы структуры](<struct.md#Методы структуры>). Если сигнатуры методов класса совпадают с теми, что указаны в интерфейсе - класс начинает удовлетворять этому интерфейсу:
```go
package main

import "fmt"

type Duck interface {
	Talk() string
	Walk()
	Swim()
}

type Dog struct{}

func (d Dog) Talk() string {
	return "AAAGGGRRR"
}

func (d Dog) Walk() {
	fmt.Println("Walking...")
}

func (d Dog) Swim() {
	fmt.Println("Swimming...")
}

func CatchDuck(duck Duck) {
	fmt.Println("Catching...")
}

func main() {
	dog := Dog{}
	CatchDuck(dog) // OK
}
```
# Копирование при присваивании
При присваивании значения интерфейсу, объект **копируется** внутрь интерфейса.
```go
package main

import (
	"fmt"
	"unsafe"
)

type eface struct {
	typ unsafe.Pointer
	val unsafe.Pointer
}

func main() {
	var value int = 100
	var i interface{} = value
	fmt.Println(i) // 100

	value = 200
	fmt.Println(i) // 100

	obj := (*eface)(unsafe.Pointer(&i))
	println("&value:", &value)   // 0x1400007cef8, on stack
	println("obj.val:", obj.val) // 0x100e6bc60, on heap

}
```
# Static and dynamic type
Переменная всегда имеет **статический тип**, даже если этот тип - интерфейс. Но [внутри](<#Header>) интерфейс хранит информацию **о динамическом типе**, который его реализует. 
```go
package main

import (
	"time"
)

type Stringer interface {
	String() string
}

func main() {
	var s Stringer  // static type
	s = time.Time{} // dynamic type
	//s = Data{}    // dynamic type
	_ = s
}
```
# Static and dynamic dispatch
**Static dispatch** - когда тип экземпляра класса для вызываемого метода известен, и мы знаем какую функцию вызвать.
**Dynamic dispatch** - когда тип экземпляра для вызываемого метода неизвестен заранее, нужно найти способ вызвать правильный метод у него.

У статического типа `&Square` есть 2 метода - `Area` и `Perimeter`. И вызов методов у экземпляра очевиден - это static dispatch.
В конструктор `NewInterface` мы принимаем статический тип. И биндим его методы в таблицу методов интерфейса (у нас это массив функций). Теперь при вызове методов у `Interface` мы вызываем методы динамического (underlying) типа `*Square`. Но мы так же можем передать и другой тип, у которого есть такие же методы. 
```go
package main

import "fmt"

type Square struct{}

func (s *Square) Area() float64 {
	return 0.0
}

func (s *Square) Perimeter() float64 {
	return 0.0
}

// ----------------------------------

const (
	AreaMethod = iota
	PerimeterMethod
)

type Interface struct {
	methods [2]func() float64
}

func NewInterface(square *Square) Interface {
	return Interface{
		methods: [2]func() float64{
			AreaMethod:      square.Area,
			PerimeterMethod: square.Perimeter,
		},
	}
}

func (i *Interface) Area() float64 {
	return i.methods[AreaMethod]()
}

func (i *Interface) Perimeter() float64 {
	return i.methods[PerimeterMethod]()
}

func main() {
	// static dispatch
	square := &Square{}
	fmt.Println(square.Area())
	fmt.Println(square.Perimeter())

	// dynamic dispatch
	iface := NewInterface(square)
	fmt.Println(iface.Area())
	fmt.Println(iface.Perimeter())
}
```
# Devirtualization
[Source code](https://go.dev/src/cmd/compile/internal/devirtualize/devirtualize.go)
Девиртуализация (приведение интерфейса к правильному типу) перед вызовом метода имеет такую же производительность, как и вызвов метода у статического типа:
```go
package main

import "testing"

// go test -bench=. -benchmem main_test.go

type ValueInterface interface {
	Add() int
}
type Value struct {
	number int
}

//go:noinline
func (v Value) Add() int {
	return v.number + v.number
}

var Result int

func Get(index int) ValueInterface {
	if index == 0 {
		return Value{}
	} else {
		panic("cant happen")
	}
}

// 1000000000               0.6987 ns/op          0 B/op          0 allocs/op
func BenchmarkDirect(b *testing.B) {
	var value Value
	for i := 0; i < b.N; i++ {
		Result = value.Add() // static dispatch
	}
}

// 1000000000               0.6934 ns/op          0 B/op          0 allocs/op
func BenchmarkWithInterface(b *testing.B) {
	var iface ValueInterface = Get(0) // hack to prevent optimisations
	for i := 0; i < b.N; i++ {
		Result = iface.(Value).Add() // devirtualisation
	}
}

```
# Empty interface (any)
Может принимать любой динамический тип, так у этого интерфейса нет методов.
Вместо указателя на `*itab` - указатель на динамический тип.
```go
type eface {
	_type *_type        // pointer to dynamic type
	data unsafe.Pointer // pointer to data
}
```
## Размер
Размер пустого интерфейса:
- 32bit - 8 byte
- 64bit - 16 byte
```go
func main() {
	var i any
	println(unsafe.Sizeof(i)) // 16byte
}
```
## Сравнивание
Нужно помнить, что есть [[uncomparable types]] типы, для которых не определен оператор `==`.
Напомним, что [[slice#Slice is not comparable (cant use as key in map )]] и является одним из таких. Но он удовлетворяет типу `any (пустой интерфейс)`. На этапе компиляции мы не узнаем об ошибке. Но в рантайме, при сравнении слайсов, произойдет [паника](<panic и recover.md#panic>).
```go
package main

import "fmt"

func main() {
	var lhs interface{} = []int{1, 2, 3}
	var rhs interface{} = []int{1, 2, 3}
	fmt.Println(lhs == rhs) // panic: comparing uncomparable type []int
}
```

# Type assertion
## Определение
`variable` должна иметь статический тип интерфейса, тогда мы сможем использовать type assertion.
Можно использовать синтаксис `val, ok := variable.(Type)` чтобы вытащить из интерфейса его динамический тип или привести один интерфейс к другому. Если статический `Type` указан правильный, то значение динамического **копируется в переменную** и флаг `ok` будет true.  Если нет - [[nil]] и `false`.
Важно понимать, что проверка происходит в рантайме, c использованием [[Reflection]]. Если не использовать флаг `ok`, то будет паника.
```go
package main

import "fmt"

func main() {
	var value int = 100
	var i interface{} = value

	conv1, ok1 := i.(int) // true, copying data from iface into converted1
	if ok1 {
		fmt.Println("conv1 int:", conv1)
	}

	conv2, ok2 := i.(float32) // false, but no panic
	if ok2 {
		fmt.Println("conv2 float32:", conv2)
	}

	conv3 := i.(int)
	fmt.Println("conv3 int:", conv3)
	conv4 := i.(float32)       // panic in runtime
	fmt.Println("conv4 float32:", conv4)
}
```
## Приведение интерфейса к другому
`variable.(Type)` проверяет, что `variable != nil`, так как [[#Empty interface (any)]] может не иметь динамического типа и значения. 
И если:
- `Type` не интерфейс (int) - проверяет что динамический тип `variable` это `Type`
- `Type` интерфейс (Stringer) - проверяет что динамический тип `variable` его реализует

Важно понимать, что при успешном type assertion одного интерфейса к другому, значение динамического типа **СОХРАНЯЕТСЯ** и **КОПИРУЕТСЯ** в новую переменную:
```go
package main

type ABC inteface {
	A()
	B()
	C()
}
type AB inteface {
	A()
	B()
}
type BC inteface {
	A()
	B()
	C()
}

type abc struct{}
func (a abc) A(){}
func (a abc) B(){}
func (a abc) C(){}

func main() {
	var a interface{}
	a = abc{}
   
	ab := a.(AB)     // type = abc, interface = AB
	ab.A()           // OK

	bc := ab.(BC)    // type = abc, interface = BC
	bc.C()           // OK

	abc1 := bc.(ABC) // type = abc, inteface = ABC
	abc1.A()         // All OK
	abc1.B()
	abc1.C()
}
```
## Анонимные интерфейсы
Мы можем приводить к анонимному интерфейсу с методами:
```go
package main

import "fmt"

type fooer interface{ foo() }

type thing struct{}
func (t *thing) foo() {}
func (t *thing) bar() {}

func main() {
	var i fooer = &thing{}

	// dynamically checking for certain methods
	_, ok := i.(interface{ bar() })
	fmt.Println("result:", ok)
}
```
# Type switch
## Определение
Можно использовать [[#Type assertion]] вместе со `switch`. Тогда вариантами выступают типы:
```go
package main

import "fmt"

func do(i interface{}) {
	switch v := i.(type) {
	case int:
		fmt.Println("int type:", v)
	case string, fmt.Stringer:
		fmt.Println("string type:", v)
	default:
		fmt.Println("unknown type:", v)
	}
}

func main() {
	do(21)
	do("hello")
	do(true)
}
```
## Порядок выбора
Класс `thing` реализует все 3 интерфейса из switch. В `switch` будет выбрано первое совпадение по порядку:
```go
package main

import "fmt"

type fooer interface{ foo() }
type barer interface{ bar() }
type foobarer interface {
	foo()
	bar()
}

type thing struct{}

func (t *thing) foo() {}
func (t *thing) bar() {}

var i foobarer = &thing{}

func main() {
	switch v := i.(type) {
	case fooer:
		fmt.Println("fooer:", v) // first match -> return here
	case barer:
		fmt.Println("barer:", v)
	case foobarer:
		fmt.Println("foobarer:", v)
	default:
		panic("none of them")
	}
}
```
# Interface guard
Можно проверить удовлетворяет ли тип интерфейсу на этапе компиляции:
```go
package main

import "io"

type T struct {
	//...
}

// Interface guard
var _ io.ReadWriter = (*T)(nil) //compile-time, will fail if not enough methods

func (t *T) Read(p []byte) (n int, err error) {
	return 0, nil
}

func (t *T) Write(p []byte) (n int, err error) {
	return 0, nil
}
```
# Interface call
Just for fun:
```go
package main

import "fmt"

type Interface interface {
	process(int) bool
}

type String string

func (s String) process(size int) bool {
	return len(s) > size
}

func main() {
	var i String = String("inteface")
	fmt.Println(i.process(10))
	fmt.Println(Interface.process(i, 10))
	fmt.Println(interface{ process(int) bool }.process(i, 10))
}
```
# Конвертация слайса интерфейсов
Можно через `for..range`, можно через [[Unsafe package]].  По сути нужно поменять тип `ReadWriter` на `Reader`, и это безопасно.
```go
package main

import "io"

func convert(a []io.ReadWriter) []io.Reader {
	var result []io.Reader
    
	for _, v := range a {
		result = append(result, v) // slice copying
	}
       
	return result
}

func readAll(a []io.Reader) {} // Only Read method

func main() {
	var a = []io.ReadWriter{} // Read + Write methods
    
	// readAll(a)                                  // compile error
	readAll(convert(a))                            // OK, slow
	readAll(*(*[]io.Reader)((unsafe.Pointer)(&a))) // OK, fast
} 
```
# Определение
Программная единица, позволяет хранить и обрабатывать однотипные и/или логически связанные данные:
```go
type Person struct {
	name, surname string
	age int
}
```

Копирование структур:
```go
func main() {
	d1 := Person{"ivan", "ivanov", 12}
	d2 := d1 // Копирование данных
}
```
# Запретить неименованую инциализацию полей
Если первым полем добавить неименованое поле типа [пустая структура](#Пустая%20структура) (занимает 0 байт), то мы не сможем инциализировать структуру без имен полей.

**ВАЖНО**: в силу [обработки пустой структуры](#Пустая%20структура##0%20байт%20всегда), если анонимное поле поставить в конец структуры - компилятор добавит 8 байт к размеру (1 байт структуры и 7 для [выравнивания](#Allignment). Поэтому поле нужно ставить **ПЕРВЫМ** по списку. Оно будет иметь такой же адрес, как и ненулевое поле после него.
```go
package main
type Config struct {
    _    struct{} // 0 byte, but ONLY AS FIRST FIELD
    Host string   
    Port int64    
}
func main() {
	с1 := Config{Host: "localhost", Port: 8080} // OK
	_ = с1

	с2 := Config{"localhost", 8080}             // compile time error
	_ = c2
}
```
# Область видимости полей ([Инкапсуляция](Инкапсуляция.md))
Переменные и поля структуры могут быть публичными и приватными.
```go
type Person {
	Name, Surname string  // public  fields
	guiltyPleasure string // private field
	ばか bool              // private field
}
```
Если первая буква имени поля - то оно
- Заглавная - публичное, доступно везде
- маленькая -  приватное, доступно в пределах package
- **иероглиф** - приватное!!!
Замена [Инкапсуляция](Инкапсуляция.md) из [1.1 OOP](1.1%20OOP.md).
# Встраивание типов ([Наследование](Наследование.md))
## Типа наследование
Встраивание внешних типов в структуру позволяет использовать их как анонимные поля. Автоматически наследуются их методы и свойства. Инициализация происходит через название типа. 
Является заменой [Наследование](Наследование.md) из [1.1 OOP](1.1%20OOP.md).
```go
type Event struct {
    ID int
    time.Time
}

func main() {
    event := Event{ID: 1234, Time: time.Now()}
    println(event.ID, event.Unix())
}
```
## Перекрытие методов
Можно перекрывать методы при встраивании, те родительный и встраиваемый типы могут иметь одинаковые методы:
```go
package main

import "fmt"

type Person struct {
	Name string
}

func (p *Person) Intro() string {
	return p.Name
}

type Woman struct {
	Person
}

func (w *Woman) Intro() string {
	return "Mrs. " + w.Person.Intro()
}

func main() {
	woman := Woman{
		Person: Person{
			Name: "Ekaterina",
		},
	}

	fmt.Println(woman.Intro())        // Mrs. Ekaterina, перекрываем метод
	fmt.Println(woman.Person.Intro()) // Ekaterina
}
```

## Рекурсивное встраивание
Нельзя рекурсивно встраивать себя, так как на этапе компиляции нужно знать размер структуры. Но можно использовать указатели:
```go
type Data struct {
	Data    // compile error
}

type Data struct {
	d *Data // OK
}
```
# Методы структуры
У структур можно объявлять методы.
Сначала мы определяем для какого типа будет этот метод (значение/указатель), затем идет стандартная сигнатура [function](function.md):

`p Person` имеет название **Receiver**. Является аналогом `this` из других языков. С помощью ресивера мы можем обращаться к структуре (классу) внутри функции (метода):
```go
type Person struct{
	age int
}
func (p Person) Age() {        // Значение Person копируется на стек функции
	return p.age
}
func (p *Person) HappyBday() { // Копируется только указатель на Person
	p.age += 1
}
```

На самом деле, это синтаксический сахар и компилятор разворачивает код в такой вид:
```go
func Age(p Person){
	return p.age
}
func HappyBday(p *Person){
	p.age +=1
}
```

Доказательство:
```go
package main

import "fmt"

type Data struct{}

func (d Data) Print() {
	fmt.Println("data")
}

func main() {
	var data Data

	data.Print()         // data
	(&data).Print()      // data

	(Data).Print(data)   // data
	(*Data).Print(&data) // data
}
```
Прочтите чем отличается [ресивер по значению и по указателю](Ресивер%20по%20значению%20и%20по%20указателю.md).
# Анонимные структуры
Самое частое применение для объявления тест-кейсов.
## Простое создание
```go
myCar := struct { 
    make, model string 
}{"tesla", "model3"}
```
## Вложенная структура
```go
person := struct {
    name string
    car struct {
        make, model string
    }
}{
    name: "Алиса",
    car: struct {
        make, model string
    }{"tesla", "model3"}
}
```
## Дофигавложенная стурктура
```go
package main

func main() {
	var book = struct {
		author struct {
			name    string
			surname string
		}
		title string
	}{
		author: struct { // обязатально нужно объявить еще раз
			name    string 
			surname string
		}{
			name:    "Name",
			surname: "Surname",
		},
		title: "Title",
	}

	_ = book
}
```
## Использование в тестах
```go
testCases := []struct {
    input    int
    expected bool
}{
    {5, true},
    {10, false},
}
```
# Union
## SBO
Предположим, что маленькие массивы мы хотим аллоцировать на [стеке горутины](Стек%20горутины.md), а большие в куче.
В случае размера > 16Byte, все данные лежат в `union`. Если размер больше - в `union` будет лежат указатель на массив и его capacity.
```go
package main

import "unsafe"

// SBO (Small Buffer Optimization)
type SBO struct {
	size  int64
	union [16]byte // 8B[pointer]8B[capacity]
}

func main() {
	var small SBO
	small.size = 10
	small.union = [16]byte{}

	var big SBO
	big.size = 1024
	pointer := (*[2048]byte)(unsafe.Pointer(&big.union))
	*pointer = [2048]byte{}
	capacity := (*int64)(unsafe.Add(unsafe.Pointer(&big.union), 8))
	*capacity = 2048
}
```
## C-style
Похоже на [slice transformation](slice#Slice%20transformation).
```go
package union

import "unsafe"

type Union struct{
	value [8]byte
}

func (u *Union) SetInt64(value int64) {
	*(*int64)(unsafe.Pointer(&u.value)) = value
}

func (u *Union) GetInt64() int64 {
	return *(*int64)(unsafe.Pointer(&u.value))
}

func (u *Union) SetFloat64(value float64) {
	*(*float64)(unsafe.Pointer(&u.value)) = value
}

func (u *Union) GetFloat64() float64 {
	return *(*float64)(unsafe.Pointer(&u.value))
}
```
# Тэги полей
У полей могут быть теки (набор пар ключ-значение). Они опциональны, парсятся в AST и нужны для правильного парсинга в разных форматов в Гошные структуры([8. Reflection](8.%20Reflection.md)):

```go
type User struct{
	Name string `json:"name" xml:"name"`
	Surname string `json:"surname" xml:"surname"`
}
```
# Псевдонимы и определения
- Type alias - не создает новый тип, просто псевдоним
- Type definition - создает новый тип относительно underlying type
```go
package main

type (
	A = int // type allias
 	B int   // type definition
)

func main() {
	var a A = 1
	var b B = 2

	var ia int = a // OK
	_= ia

	var ib int = b // compile error, need to cast int(b)
	_ = ib
}
```

С помощью [unsafe](6.%20Unsafe%20package.md) можно кастит type definition к underlying type:
```go
package main

import (
	"testing"
	"unsafe"
)

// go test -bench=. -benchmem performance_test.go

type Int int

var convertedData []int

// 833.1 ns/op
func BenchmarkCast(b *testing.B) {
	data := make([]Int, 1024)
	for i := 0; i < b.N; i++ {
		convertedData = make([]int, 1024)
		for idx, value := range data {
			convertedData[idx] = int(value) // safe cast in for loop
		}
	}
}

// 0.55 ns/op
func BenchmarkUnsafeCast(b *testing.B) {
	data := make([]Int, 1024)
	for i := 0; i < b.N; i++ {
		convertedData = *(*[]int)(unsafe.Pointer(&data)) // cast to underlying
	}
}
```

# Padding 
## False sharing
[Padding is hard](https://dave.cheney.net/2015/10/09/padding-is-hard)
[False sharing](https://habr.com/ru/companies/vk/articles/510200/)

**False sharing** возникает, когда два независимых поля, к которым обращаются разные горутины, попадают в **одну и ту же cache line** (обычно 64 байта).

Даже если потоки работают с разными переменными, процессор вынужден постоянно **инвалидировать** и **перезагружать** одну и ту же cache line, потому что каждая запись помечает её как _modified (M-state, MESI)_. В результате резко растут задержки из-за межъядерной синхронизации.
```go
type Counter struct {     
	A int64 // обновляет одна горутина     
	B int64 // обновляет другая, но обе попадают в одну cache line 
}
```
## Решение: добавить padding, чтобы поля оказались в разных cache lines
С помощью анонимных полей, мы можем добить поля до размера cache line.
```go
type PaddedCounter struct {     
	A int64     
	_ [56]byte // добиваем до 64 байт, чтобы изолировать cache line      
	B int64     
	_ [56]byte // изолируем вторую 
}
```

Теперь каждое поле занимает **свою собственную cache line**, и никакой межъядерной конкуренции по протоколу MESI больше не возникает.
```
                  ┌──────────────────────────────────────────┐        
                  │                 Main memory              │        
                  │                                          │        
                  │ ┌────────┌──────────┐────────┌─────────┐ │        
                  │ │  var1  │ padding  │  var2  │ padding │ │        
                  │ └────────└──────────┘────────└─────────┘ │        
                  └──────────────────────────────────────────┘        
                                                                      
                                                                      
┌────────────────────────────────┐  ┌────────────────────────────────┐
│             Core 1             │  │             Core 2             │
│ ┌────────────────────────────┐ │  │ ┌────────────────────────────┐ │
│ │             L1             │ │  │ │             L1             │ │
│ │┌────────┌─────────────────┐│ │  │ │┌────────┌─────────────────┐│ │
│ ││  var1  │     padding     ││ │  │ ││  var2  │     padding     ││ │
│ │└────────└─────────────────┘│ │  │ │└────────└─────────────────┘│ │
│ └────────────────────────────┘ │  │ └────────────────────────────┘ │
└────────────────────────────────┘  └────────────────────────────────┘
```
# Allignment 
## Определение
В Go каждый примитивный тип данных должен начинаться в памяти по определённой границе. Это называется выравниванием (alignment).
```
type                 allignment guarantee
-----                -----
bool, uint8, int8    1
uint16, int16        2
uint32, int32        4
float32, complex64   4
arrays               depend on element types
structs              depend of field types
other types          native word
```
Иными словами, значение int32 должно находиться по адресу, кратному 4, значение int16 – по адресу, кратному 2, и так далее. Компллятор делает это по умолчанию.
## Почему
Современные CPU могут читать и писать данные размером в машинное слово атомарно, если адрес значения выровнен. Если же, например, на 32-битной системе значение int32 расположено «криво» – по адресу, кратному 2, но не кратному 4 – процессору приходится выполнять две операции чтения или записи вместо одной.

За выравнивание приходится платить: размещая данные на ровных границах, Go иногда добавляет лишние байты паддинга, которые не используются.

В этом примере, самое большое поле структуры - int32, занимает 4 байта. Значит каждое поле будет выровнено до четырёх байт. И порядок размещения полей может влиять на итоговый размер структурыю Узнать размер структуры можно, используя [unsafe.Sizeof](Sizeof.md):
```go
package main

import (
	"fmt"
	"unsafe"
)

/*
1byte  2      3      4      5      6      7      8byte
+------+------+------+------+------+------+------+------+
| bool |     alignment      |        int32              |
+------+------+------+------+------+------+------+------+
| bool |     alignment      |   
+------+------+------+------+
9      10     11     12    
*/
type data1 struct {
	aaa bool  // 1byte
	// _ [3]byte
	bbb int32 // 4 byte
	ccc bool  // 1 byte
	// _ [3]byte
}

/*
1byte  2      3      4      5      6      7      8 byte
+------+------+------+------+------+------+------+------+
|         int32             | bool | bool | allignment  |
+------+------+------+------+------+------+------+------+
*/
type data2 struct {
	aaa int32 // 4 byte
	bbb bool  // 1 byte
	ccc bool  // 1 byte
	// _ [2]byte
}

func main() {
	fmt.Println(unsafe.Sizeof(data1{})) // 12
	fmt.Println(unsafe.Sizeof(data2{})) //  8

	/*
		d := data1{
			aaa: true,
			bbb: 5,
			ccc: true,
		}

		b := (*[12]byte)(unsafe.Pointer(&d))
		fmt.Printf("Bytes are %#v\n", b)
	*/
}
```
# Проверить размер на этапе компиляции
Проверять размер можно в рантайме, можно на этапе компиляции (перед запуском программы). Функции `sizeof`, `offsetof`, `alignof` возвращают константы, известные на этапе компиляции.
Компилятор не даст переполнить беззнаковый `uintptr`, если структура больше нужного размера:
```go
package main

import "unsafe"

type Data struct {
	id   int32
	data [60]byte
}

// Describe:
// - Alignof
// - Offsetof

var _ uintptr = unsafe.Sizeof(Data{}) - 64
var _ uintptr = 64 - unsafe.Sizeof(Data{})
```
# Сравнение структур
Порядок полей влияет на эффективность сравнения стуктур. Оно начинается сверху вниз, поле с полем. В случае `Data1`, если отличаются поля `size` (integer) - сравнения дальше **не будет**. Если же сначала сравнивать большие поля - просадка по производительности существенная:
```go
package main

import (
	"testing"
)

// go test -bench=. comparison_test.go

type Data1 struct {
	size   int32
	values [10 << 20]byte
}

type Data2 struct {
	values [10 << 20]byte
	size   int32
}

func BenchmarkComparisonData1(b *testing.B) {
	data1 := Data1{size: 100}
	data2 := Data1{size: 101}
	for i := 0; i < b.N; i++ {
		_ = data1 == data2 // 1.507 ns/op
	}
}

func BenchmarkComparisonData2(b *testing.B) {
	data1 := Data2{size: 100}
	data2 := Data2{size: 101}
	for i := 0; i < b.N; i++ {
		_ = data1 == data2 // 211875 ns/op
	}
}
```
# Пустая структура
## 0 байт всегда
До версии `go 1.5`, struct{} всегда занимала 0байт, но обладала собственным адресом. 

После версии `go 1.5` все изменилось. У любого поля структуры можно взять адрес. Так как `struct{}` занимает 0байт, то взяв у нее адрес мы могли бы адресоваться за пределы структуры. А если мы можем указывать за пределы структуры - можно посадить утечку памяти (когда непредномеренно сохраняется указатель на другой участок памяти и [5. GC](5.%20GC.md) не может его освободить). Чтобы избежать такого поведения, компилятор неявно добавит 1 байт к `struct{}` + [allignment](#Allignment).
```go
package main

import "fmt"

func main() {
	s := new(struct {
		i int
		z struct{}
	})
	i := &s.i          // 0x14000106020
	a := &s.z          // 0x14000106028
	b := new(struct{}) // 0x102514980
	fmt.Printf("%p %p %p\n", i, a, b)
}
```

То есть в случае, если struct{} идёт последним полем, его адрес "вылазит" за границу структуры — и компилятору приходится расширить структуру, чтобы этот выход стал корректным и чтобы указатель на пустую структуру не указывал "в никуда" за пределы внешней структуры.
## zerobase - единый адрес на всех
Все переменные типа `struct{}` без полей получают адрес &zerobase - скрытая глобальная переменная в рантайме [mallocgc](https://go.dev/src/runtime/malloc.go) :
```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if doubleCheckMalloc {
		if gcphase == _GCmarktermination {
			throw("mallocgc called with gcphase == _GCmarktermination")
		}
	}

	// Short-circuit zero-sized allocation requests.
	if size == 0 {
		return unsafe.Pointer(&zerobase)
	}
```

Поэтому у всех пустых структур в коде один адрес `&zerobase`. Это гарантирует, что даже если есть множество разных значений struct{}, их адреса будут указывать на одно и то же место — тем самым мы снижаем накладные расходы, но сохраняя семантику "адресуемости" даже пустой переменной.
## Приколы со [cтеком горутины](Стек%20горутины.md)
В стеке Go может дать разным переменным разные адреса, даже если они zero-sized.
Переменные `a` и `b` лежат на [стеке горутины](Стек%20горутины.md) main, но проверка равности их адресов возвращает false.

А для `s1` и `s2` произошел перенос в кучу за счет Printf -> применилась оптимизация и переменным был присвоен общий фиктивный [zerobase](#zerobase%20-%20единый%20адрес%20на%20всех) адрес.
```go
package main

import "fmt"

func leak() *struct{} {
	s := struct{}{}
	return &s
}

func main() {
	s1 := leak()
	s2 := leak()
       
	a := struct{}{}
	b := struct{}{}
      
	fmt.Println(&a == &b)         // false, a,b on stack
	// println(&a, &b)            // 0x14000116ef7 0x14000116ef7; on stack
	// fmt.Println("a,b", &a, &b) // if uncomment - a,b will escape to heap
	
	fmt.Println(s1 == s2)         // true
	fmt.Println("s1,s2", s1, s2)  // 0x100b88980, 0x100b88980; on heap 
}
```
На самом деле, false при сравнении адресов - это [hardcode](https://www.reddit.com/r/golang/comments/orfjdr/comment/h6hux2h/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button). На этапе компиляции подставляется `false` и возвращается. Скорее всего, какая-то оптимизация.

Интервьюер может спросить:
```go
arr := [6]*struct{}{&a} and 
brr := [6]*struct{}{&b}, 
//why arr == brr can differ from &a == &b. 
```

  Поскольку сравнение [array](array.md) использует семантику равенства указателей, оно может вернуть значение true, если считает указатели равными, даже если соотношение &a == &b непредсказуемо. Ключевой вывод: **равенство указателей на пустые структуры изначально ненадёжно**.
### arm64
```asm
CALL    runtime.printlock(SB)
MOVW    $0, R0
MOVB    R0, 4(R13)
CALL    runtime.printbool(SB)
CALL    runtime.printnl(SB)
CALL    runtime.printunlock(SB)
```
### amd64
 _According to the assembly output from the compiler, the comparison is hardcoded to false. Speculation: This is a compiler optimization allowing for constant propagation. If it's optimized to a constant, that information can be used to perform other optimizations. Why false instead of true? Because it's the safer result to assume.
```go
package foo

func foo() bool {
        var a, b struct{}
        return &a == &b
}

$ GOOS=linux GOARCH=amd64 go tool compile -S x.go 
"".foo STEXT nosplit size=3 args=0x0 locals=0x0 funcid=0x0
        ...
        0x0000 00000 (x.go:5)   XORL    AX, AX
        0x0002 00002 (x.go:5)   RET
        ...
```
### Что ответить на интервью
Интервьюер может спросить:
```go
var a, b struct{}  
fmt.Println(&a == &b) // could be true or false, unpredictable
arr := [6]*struct{}{&a}
brr := [6]*struct{}{&b} 
//why arr == brr can differ from &a == &b. 
```
  Поскольку сравнение [array](array.md) использует семантику равенства указателей, оно может вернуть значение true, если считает указатели равными, даже если соотношение &a == &b непредсказуемо. Ключевой вывод: **равенство указателей на пустые структуры изначально ненадёжно**.

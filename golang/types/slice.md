# Header
```go
type SliceHeader struct {
 Data uintptr // Указатель на первый элемент базового массива
 Len  int     // Длина массива
 Cap  int     // Максимальное кол-во элементов в массиве
}
```
Легковесная структура, которая занимает всего 3 машинных слова. 
Дешев для [Passing by value](Passing%20by%20value.md).
# len() and cap() are builtins
 `len()/cap()` никакая не функция, а builtin. Компилятор преобразует каждый вызов len(s)/cap(s) в прямое обращение к свойству среза `s.len`/`s.cap`. Вызова функции в рантайме не происходит, и дополнительных затрат не создается.
# Инициализация
```go
var s1 []int            // nil
var s2 []int(nil)       // nil
s3 := []int{}           // []             len=0 cap=2
s4 := []int{1,2}        // [1 2]          len=2 cap=2
s5 := []int{5:10}       // [0 0 0 0 0 10] len=6 cap=6
s6 := make([]int, 6)    // [0 0 0 0 0 0]  len=6 cap=6
s7 := make([]int, 3, 6) // [0 0 0 0 0 0]  len=3 cap=6
```
# Операции с nil slice
- Чтение - panic (out of range)
- Запись - panic (out of range)
- append - ok (создается новый массив)
- for range - ok
# Итерация
```go
package main

import (
	"fmt"
)

// Доступ за пределы длины
func accessToElement1() {
	data := make([]int, 3)
	fmt.Println(data[4]) // panic
}

// Доступ за пределы длины, в пределах капасити
func accessToElement2() {
	data := make([]int, 3, 6)
	fmt.Println(data[4]) // panic
}

func accessToElement3() {
	data := make([]int, 3, 6)
	_ = data[-1] // compilation error
}

func makeZeroSlice() {
	data := make([]int, 0)
	fmt.Println(len(data)) // 0
	fmt.Println(cap(data)) // 0
}

func makeSlice() {
	_ = make([]int, -5)    // compilation error
	_ = make([]int, 10, 5) // compilation error

	size := -5
	_ = make([]int, size) // panic

	size = 5
	_ = make([]int, size*2, size) // panic
}
```
# Append
Функция append принимает variadic параметры. В реальности, это слайс соответствующего типа. Этот слайс может содержать, может не содержать значения. Поэтому и кол-во параметров в append может быть нулевым. 
```go
// arr = [1,2,3,4]
arr = append(arr)                     // no changes
arr = append(arr, 5)                  // [1,2,3,4,5]
arr = append(arr,5,6,7)               // [1,2,3,4,5,6,7]
arr = append(arr, arr, []int{5,6}...) // [1,2,3,4,5,6]
```

Append **ВСЕГДА** возвращает новый срез - с обновленный len, cap и указателем, если была реаллокация.
## Реализация
```go
package main

import "fmt"

func Append[T any](slice []T, elems ...T) []T {
	if len(elems) == 0 {
		return slice
	}
    
	prevLen := len(slice)
	newLen := prevLen + len(elems)
    
	if newLen > cap(slice) {
		// Growth strategy: double the capacity, 
		// but ensure it's at least newLen.
		newCap := max(cap(slice) * 2, newLen)
		if newCap == 0 {
			newCap = newLen
		}
		
		newSlice := make([]T, newLen, newCap)
		copy(newSlice, slice)
		slice = newSlice
	} else {
		slice = slice[:newLen]
	}
      
	copy(slice[prevLen:newLen], elems)
	
	return slice
}
```
## Инициализация нечетным слайсом
Если пустой слайс инциализируется другим с **НЕЧЕТНОЙ** длиной и она больше трёх, то у результирующего capacity будет **НА ОДИН БОЛЬШЕ** длины. 
Если длина четная - capacity будет равен длине.
```go
arr := []int{} // len=0, cap=0

arr = append(arr, []int{1,2,3,4,5,6}...)  // len=6, cap=6
// BUT!!!!!!
arr = append(arr, []int{1,2,3,4,5,6,7}...) // len=7, cap=8 (WTF???)
```
[Задача](#задача3)
## Формула роста capacity
Если достигли `capacity`, при добавлении, append создаст новый массив:
src/runtime/slice.go
![Pasted image 20251130142816.png](Pasted%20image 20251130142816.png.md)
[Задача №1](#задача2) 
[Задача №2](#задача4)
[Задача №3](#задача6)
# Reduce slice capacity (уменьшить память)
Перевыделяем память, копируем нужные элементы. Так как перестаем ссылаться на изначальный массив, [5. GC](5.%20GC.md) его освободит.
```go
data := []int{1,2,3,4,5}
data = append([]int(nil), data[:2]...)
/*
+------------+
|   data     | --/> [1 2 3 4 5]
+------------+ 
|   len(3)   | ---> [1 2] 
+------------+    
|   cap(6)   | 
+------------+   
*/
```
# Range over slice
![Pasted image 20251130175815.png](Pasted%20image 20251130175815.png.md)
Поэтому, range over slice - это дешевая операция. А [array#Range over array](array#Range%20over%20array.md) - нет!
[Задача](#задача1)
# Subslice
```
data1 := make([]int, 3, 6)   +------------+
data2 := data1[1:3]          |   data1    | -> [0 {0 0}],0,0,0
							 +------------+       ^
							 |   len(3)   |       |
							 +------------+       |
							 |   cap(6)   |       |
							 +------------+       |
                                                  |
							             +------------+
										 |   data2    |
										 +------------+
										 |   len(2)   |
										 +------------|
										 |   cap(5)   |
										 +------------+

data1 := make([]int, 3, 6)   +------------+
data2 := data1[1:3]          |   data1    | -> [0 {1 0}],0,0,0
data[1]=1				     +------------+       ^
							 |   len(3)   |       |
							 +------------+       |
							 |   cap(6)   |       |
							 +------------+       |
                                                  |
							             +------------+
										 |   data2    |
										 +------------+
										 |   len(2)   |
										 +------------|
										 |   cap(5)   |
										 +------------+

data1 := make([]int, 3, 6)   +------------+
data2 := data1[1:3]          |   data1    | -> [0 {1 0] 2},0,0
data[1]=1				     +------------+       ^
data2=append(data2, 2)	     |   len(3)   |       |
							 +------------+       |
							 |   cap(6)   |       |
							 +------------+       |
                                                  |
							             +------------+
										 |   data2    |
										 +------------+
										 |   len(3)   |
										 +------------|
										 |   cap(5)   |
										 +------------+

data1 := make([]int, 3, 6)   +------------+
data2 := data1[1:3]          |   data1    | -> [0 {1 0 3}],0,0
data[1]=1				     +------------+       ^
data2=append(data2, 2)	     |   len(4)   |       |
data1=append(data1,3)		 +------------+       |
							 |   cap(6)   |       |
							 +------------+       |
                                                  |
							             +------------+
										 |   data2    |
										 +------------+
										 |   len(3)   |
										 +------------|
										 |   cap(5)   |
										 +------------+
```
[Задача №1](#задача4)
[Задача №2](#задача5)
[Задача №3](#задача6)
# Утечки памяти
Если меньшие сабслайсы ссылаются на большие слайсы, то огромный массив под ними не будет освобожден пока все ссылки станут невалидными. Это значит, что даже если нам не нужен больший слайс, то сабслайс все равно будет держать ссылку на этот участок памяти и не давать [5. GC](5.%20GC.md) освободить эту память.
```go
package main

import "fmt"

func printAllocs() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("%d MB\n", m.Alloc/1024/1024)
}

func getBytes(start,end int) []byte {
	arr := make([]byte, 1<<30)
	slice := arr[start:end]
	return slice
}

func main() {
	s := getBytes(10, 20)     //from 10 to 20 byte
	println("cap = ", cap(s)) //1073741814
	fmt.Println(s)            //[0 0 0 0 0 0 0 0 0 0]

	printAllocs()             //1024 MB
	runtime.GC()
	printAllocs()             //1024 MB

	runtime.KeepAlive(s)
}
```

Такое же поведение если ссылаться на единственный элемент большого массива:
```go
package main

import (
	"fmt"
	"runtime"
)

func printAllocs() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("%d MB\n", m.Alloc/1024/1024)
}

func findElem(numbers []int, target int) *int {
	//...
	return &numbers[0]
}

func main() {
	arr := make([]int, 1<<30)
	ptr := findElem(arr, 0)
	_ = ptr

	printAllocs() //8192 MB
	runtime.GC()
	printAllocs() //8192 MB

	runtime.KeepAlive(ptr)
}
```

Чтобы пофиксить - возвращайте [новую копию](#slice%20copy) из функции.
# Loop unwind
```go
package main

import (
	"testing"
)

// go test -bench=. comparison_test.go

const size = 1 << 10

func BenchmarkWithoutOptimizations(b *testing.B) {
	data := make([]int, size)
	for i := 0; i < b.N; i++ {
		for j := 0; j < len(data); j++ {
			data[j] = i
		}
	}
}

func BenchmarkWithLoopUnwinding(b *testing.B) {
	data := make([]int, size)
	for i := 0; i < b.N; i++ {
		for j := 0; j < len(data)/4; j += 4 {
			data[j] = i
			data[j+1] = i
			data[j+2] = i
			data[j+3] = i
		}
	}
}
```
# Slice copy
Мы можем явно контролировать сколько элементов копировать путем задания длины dst слайса.
```go
package main

import "slices"

func main() {
	src := []int{1, 2, 3, 4, 5}
	dst := make([]int, 3)
	// First
	copy(dst, src)

	// Second
	dst = append([]int(nil), src...)

	// Third
	dst = slices.Clone(src)
}
```
# Dirty slice
Когда создается срез, все его элементы инициализируются с [1.3 zero value](1.3%20zero%20value.md) и занимают память. 
Но иногда нам нужно создать большой срез, на несколько ГБ без инициализации.

```go
package main

import (
	"strings"
	"testing"
	"unsafe"
)

// go test -bench=. main_test.go

func makeDirty(size int) []byte {
	var sb strings.Builder
	sb.Grow(size)

	pointer := unsafe.StringData(sb.String())
	return unsafe.Slice(pointer, size)
}

var Result []byte

func BenchmarkMake(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Result = make([]byte, 0, 10<<20)
	}
}

func BenchmarkMakeDirty(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Result = makeDirty(10 << 20)
	}
}
```

# Slice transformation
Мы можем трансформировать `[]byte` в слайс нужного типа. В данном примере, в изначальном `[]byte` лежало 800 байт. После трансформирования, получим 100 int. Так как один int занимает 8 байт.
```go
package main

import (
	"unsafe"
)

type slice struct {
	data unsafe.Pointer
	len  int
	cap  int
}

func Transform(data []byte) []int {
	sliceData := unsafe.Pointer(&data[0])
	sizeType := int(unsafe.Sizeof(0))
	length := len(data) / sizeType
	
	var result []int
	resultPtr := (*slice)(unsafe.Pointer(&result)) // указатель на слайс
	resultPtr.data = sliceData // модифицируем слайс result!!!
	resultPtr.len = length     // модифицируем слайс result!!!
	resultPtr.cap = length     // модифицируем слайс result!!!
	
	return result
}

func main() {
	data := make([]byte, 800)
	converted := Transform(data)
	_ = converted // len(converted) = 100
}
```
# Slice is not comparable (cant use as key in map)
Slice это ссылочный тип, так что мы могли бы через оператор  `==` сравнивать указатели, len и cap двух слайсов, но это будет отличаться от проверки для массивов, где сравниваются именно данные, а не указатели.
Но можно использовать `reflect.DeepEqual()` и `slices.Equal()` для сравнения.

1. Элементы среза являются косвенными - они хранятся в базовом массиве, и сам срез содержит указатель на этот массив, длину и емкость. Такая косвенная структура позволяет срезу ссылаться на любую часть базового массива, что позволяет срезу содержать самого себя. Например, если срез содержит указатель на часть массива, которая, в свою очередь, может быть частью другого среза или того же среза. Ситуации, когда срезы содержат друг друга, очень сложно обрабатывать, и они могут привести к неочевидным и неэффективным алгоритмам сравнения. 
2. В силу косвенности элементов фиксированное значение среза может содержать различные элементы в разные моменты времени при изменении содержимого базового массива. Каждый ключ должен быть уникальным. Поскольку хеш-таблицы делают только поверхностные копии своих ключей - то есть копируют только указатели, а не сами данные - требуется, чтобы равенство для каждого ключа оставалось неизменным на протяжении всего времени жизни хеш-таблицы. Если мы изменили базовый массив, на который ссылается слайс - как понять, что это тот же самый слайс или другой? Никак. 
3. Для срезов можно было бы ввести проверку на "поверхностное" равенство, то есть сравнивать только указатели, длину и емкость, не сравнивая сами элементы. Это решило бы проблему с хеш-таблицами, но такая проверка будет отличаться от проверки для массивов, где сравниваются именно данные, а не указатели. Такая разная трактовка оператором `==` для срезов и массивов может ввести в заблуждение программистов, поэтому авторы языка запретили такое сравнение.
4. Поскольку [map](map.md) должна использовать неизменные хеш-значения для ключей, изменение содержимого среза приводит к несоответствию хеш-значений. Это делает невозможным корректное определение бакета, где хранится значение, связанное с этим ключом.э
# Zero length slice array points to zerobased address
```go
	data = []string{}
	fmt.Println("data = []string{}:")
	fmt.Printf("\tempty=%t nil=%t size=%d data=%p\n", len(data) == 0, data == nil, unsafe.Sizeof(data), unsafe.SliceData(data))

	data = make([]string, 0)
	fmt.Println("data = make([]string, 0):")
	fmt.Printf("\tempty=%t nil=%t size=%d data=%p\n", len(data) == 0, data == nil, unsafe.Sizeof(data), unsafe.SliceData(data))

	empty := struct{}{}
	fmt.Println("empty struct address:", unsafe.Pointer(&empty))

/*
	[]string{}:               empty=true nil=false size=24 data=0x100338980
	make([]string, 0):        empty=true nil=false size=24 data=0x100338980
	empty struct address:                                       0x100338980
*/
```
# Filter slice without allocations
```go
func FilterNoAllocations(data []int) []int {
	var k = 0
	for i, v := range data {
		if check(v) {
			data[i] = data[k]
			data[k] = v
			k++
		}
	}
	return data[:k]
}
```
# Задачи
## Задача1
- `range x` создает копию [slice header](#header) и итерируется по нему. В этой копии лежат
	- указатель на массив
	- длина
	- ёмкость
-  Так как len и cap у x равны трём, то при append внутри цикла `(x = append(x,...))`  он выделяет новый массив. НО(!!!) range **продолжает итерироваться по старому** массиву, указатель на который лежит в копии slice header.
- Поэтому изменения не будут видны в момент print()
```go
package main

/*
x    -> |{A Z C Z} | {A Z Z Z Z} | {A Z Z Z Z Z}
        |          |             |
x(c) -> |{A M C}   |             |
        0A,        1M,           2C,
*/

func main() {
	var x = []string{"A", "M", "C"}

	for i, s := range x {
		print(i, s, ",")
		x[i+1] = "M"
		x = append(x, "Z")
		x[i+1] = "Z"
	}
}
```
## Задача2
Если добавлять элементы в цикле, то len и cap растут по формуле. Это отличается от случая, когда в append передается целый массив.
```go
a := []int{}        //      len=0, cap=0
for i := range 3 {  // i=0, len=1, cap=1 
	a = append(a,i) // i=1, len=2, cap=2
}                   // i=2, len=3, cap=4

// NOT THE SAME!!!
a := []int{}                     // len=0, cap=0
a = append(a, []int{1, 2, 3}...) // len=3, cap=3
```
## Задача3
В этой задаче есть подвох. При инциализации пустого слайса другим слайсом с нечетной длиной - у начального слайса капасити будет на один больше. Из-за этого мы будем видеть изменения низлежащего массива после всех операций.
```go
package main

import "fmt"

func main() {
	a := []int{} // len=0,cap=0

	a = append(a, []int{1,2,3,4,5}...) //len=5,cap=6,a=[1,2,3,4,5],0

	println("cap(a) = ", cap(a)) //cap = 6

	b := append(a, 6) //len=6,cap=6, b=[1,2,3,4,5,6], a=[1,2,3,4,5],6
	c := append(a, 7) //len=6,cap=6, c=[1,2,3,4,5,7], b=[1,2,3,4,5,7], a=[1,2,3,4,5],7

	c[1] = 0 //c=[1,0,3,4,5,7], b=[1,0,3,4,5,7], a=[1,0,3,4,5],7

	fmt.Println("a = ", a)
	fmt.Println("b = ", b)
	fmt.Println("c = ", c)
}
```
## Задача4
Синтаксис взятия сабслайса:
- левая граница **всегда** включительна (иначе писали бы `a[-1:4]` для нулевого индекса)
- правая невключительная (удобно указывать капасити)
Если мы берем сабслайс не до конца исходного слайса, то его хвост остается жить в памяти. Длина будет такой, которую мы указали в скобках взяти сабслайса, а capacity - `cap(orig) - len(subslice)`.
```go
package main

import "fmt"

func main(){
	a := []int{1,2,3,4,5} //len=5, cap=5, a=[1,2,3,4,5]
	b := a[2:4]           //len=2, cap=3, b=[3,4],5
	c := append(b,10)     //len=3, cap=3, c=[3,4,10]; b=[3,4],10; a=[1,2,3,4,10]
	c[1] = 55             //len=3, cap=3, c=[3,55,10]; b=[3,55],10; a=[1,2,3,55,10]

	fmt.Println(a)
	fmt.Println(b)
	fmt.Println(c)
}
```
## Задача5
```go
package main

import "fmt"

func main() {
	s := make([]int,0, 5)          //len=0, cap=5
	s = append(s, 1, 2, 3)         //len=3, cap=5, s=[1,2,3],0,0

	subSlice := s[1:3]             //len=2,cap=4, sub=[2,3],0,0
	
	subSlice[0] = 99               //len=2,cap=4, sub=[99,3],0,0; s=[1,99,3],0,0
	subSlice = append(subSlice, 4) //len=3,cap=4, sub=[99,3,4],0; s=[1,99,3],4,0

	s = append(s, 5, 6, 7)        //len=6,cap=10, s=[1,99,3,5,6,7];sub=[99,3,4],0

	subSlice[1] = 100             //sub=[99,100,4],0             

	fmt.Println("s: ", s)
	fmt.Println("subSlice: ", subSlice)
}
```
## Задача6
### 6.1
```go
package main

import "fmt"

func main() {
	a := [...]int{0, 1, 2, 3} //len=4,cap=4, a=[0,1,2,3]
	x := a[:1]                //len=1,cap=4, x=[0],1,2,3
	y := a[:2]                //len=2,cap=4, y=[0,1],2,3

	x = append(x,y...)       // len=3,cap=4; x=[0,0,1],3, a=[0,0,1,3], y=[0,0]
	x = append(x,y...)       // len=5,cap=8; x=[0,0,1,0,0], a=[0,0,1,3]

	fmt.Println(a)
	fmt.Println(x)
}
```
### 6.2
```go
package main

import "fmt"

func main() {
	a := [...]int{0, 1, 2, 3} //len=4,cap=4, a=[0,1,2,3]
	x := a[:1]                //len=1,cap=4, x=[0],1,2,3      
	y := a[2:]                //len=2,cap=2, y=[2,3]

	x = append(x,y...)        // x=[0,2,3],3;   a=[0,2,3,3]; y=[3,3]
	x = append(x,y...)        // x=[0,2,3,3,3]; a=[0,2,3,3]; y=[3,3]

	fmt.Println("a = ", a)
	fmt.Println("x = ", x)
}
```
# Инициализация
```go
var arr0 [0]int              // compiles, same addr as struct{}
var arr1 [5]int              // [0 0 0 0 0]
var arr2 [2][5]int           // [[0 0 0 0 0] [0 0 0 0 0]]
arr3 := [...]int{1,2,3}      // [1 2 3]
arr4 := [5]int{1,2,3}        // [1 2 3 0 0]
arr5 := [5]{3: 4}            // [0 0 0 4 0]
arr6 := [5]int{2:5, 6, 1: 7} // [0 7 5 6 0]
```
- Если размер три точки - компилятор сам его вычислит
- В качестве длины массива можно указывать **ТОЛЬКО КОНСТАНТЫ**. Нельзя использовать `var N  = 1` -  будет ошибка компиляции.
- Программа не хранит размер массива отдельно. Она является частью типа, поэтому она должна быть известна на этапе компиляции (в таблице символов).
```go
package main
func main() {
	a1 := [3]int{1,2,3}
	a2 := [2]int{1,2}
	a1 = a2 // compilation error, beacause different types
}
```
# API
```go
value := data[5] //чтение
data[5] = 1000   //запись
len(data)        //получение длины
cap(data)        //получение ёмкости
p := &data       //подучение указателя на массива
s :=data[1:4]    //получение среза из массива
```
# Unsafe итерация по массиву
```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	const elemSize = unsafe.Sizeof(int32(0)) // 4bytes
	array := [...]int32{1, 2, 3}
	ptr := unsafe.Pointer(&array)

	first := *(*int32)(unsafe.Add(ptr, 0*elemSize)) // array[0]
	second := *(*int32)(unsafe.Add(ptr, 1*elemSize)) // array[1]
	third := *(*int32)(unsafe.Add(ptr, 2*elemSize)) // array[2]

	dangerous1 := *(*int32)(unsafe.Add(ptr, -1)) // garbage
	dangerous2 := *(*int32)(unsafe.Add(ptr, 3)) // garbage
}
```
Синтаксис доступа по индексу `array[i]` - синтаксический сахар. В реальности, он разворачивается в то, что написано сверху в примере.
# Range over array
При итерации с помощью `range`, выражение (arr) сначала **КОПИРУЕТСЯ**, затем range **ИТЕРИРУЕТСЯ** по **КОПИИ**.
Если нужно итерироваться по большому массиву:
- Итерироваться по индексам
- Передать указатель на массив
- Передать слайс от массива
```go
package main

import "fmt"

func main() {
	data := [...]int{1, 2, 3}
	for value := range data { // copy of array
		fmt.Println(value)
	}

	for i := 0; i < len(data); i++{
		// do something
	}

	for value := range &data { // not a copy of array
		fmt.Println(value)
	}

	for value := range data[:] { // not a copy of array
		fmt.Println(value)
	}
}
```

При итерации с помощью range, переменная из массива копируется во временную переменную value. Поэтому изменение этой перменной никак не скажется на изначальном массиве. 
```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	arr := [...]int{100, 200, 300}

	for idx, value := range arr {
		value += 50
		fmt.Println("value ptr = ", unsafe.Pointer(&value), "arr idx ptr = ", unsafe.Pointer(&arr[idx]))
	}

	fmt.Println("arr = ", arr)
}
```

Начиная с **go 1.22**, уникальный экземпляр будет создан для каждой итерационной перменной в каждой итерации цикла:
```
value ptr =  0x14000106020; arr idx ptr =  0x1400011c018
value ptr =  0x14000106028; arr idx ptr =  0x1400011c020
value ptr =  0x14000106040; arr idx ptr =  0x1400011c028
```

Нужно это было для того, что при создании горутин в цикле, у каждой горутины была своя копия итерационной перменной.
# Iteration over nil pointer to array
Получается, что при использовании value в range, происходит разыменование указателя по индексу. Указатель [[nil]], получаем `nil pointer dereference`.
```go
package main

import "fmt"

func main() {
	var array *[4]int // = nil

	fmt.Println("length =", len(array))
	fmt.Println("capacity =", cap(array))

	for idx := range array { // ok
		fmt.Println(idx)
	}

	for idx, value := range array { // panic
		fmt.Println(idx, value)
	}
}
```
# Index vs Ptr range benchmark
![[Pasted image 20251130132627.png]]
Итерация по индексу быстрее, чем итерация по явным указателям на стуктуры.
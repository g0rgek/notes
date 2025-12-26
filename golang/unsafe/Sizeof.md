# Размер переменной на этапе компиляции
```go
package main

import(
	"fmt"
	"unsafe"
)

func main() {
	var val1 int8 = 10
	fmt.Println("size1:", unsafe.Sizeof(val1)) // 1 byte

	var val2 int32 = 10
	fmt.Println("size1:", unsafe.Sizeof(val2)) // 4 byte
}
```
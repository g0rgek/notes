# Арифметика указателей
```go
package main

import (
	"fmt"
	"unsafe"
)
func main(){
	var value uint32 = 0xFFFFFFFF
	// bytes:            1 2 3 4

	ptr := unsafe.Pointer(value)
	oneBytePtr := (*uint8)(ptr) //0x[FF]FFFFFF

	fmt.Println("value1: ", *bytePtr) //255

	ptr = unsafe.Add(ptr, 2)     //jump 2 bytes ahead
	twoBytePtr := (*uint16)(ptr) //0xFFFF[FFFF]

	fmt.Println("value2: ", *twoBytePtr) //65535
}
```

# Прочитать поле структуры/элемент массива
```go
// sf := unsafe.Pointer(&s.f)
sf := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f))

// ai := unsafe.Pointer(&x[i])
ai := unsafe.Pointer(uintptr(unsafe.Pointer(&x[0])) + i * unsafe.Offsetof(x[0]))
```
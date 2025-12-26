# GC doesn't track uintptr pointers
```go
//go run -gcflag=-d=checkptr main.go

package main

import (
	"runtime"
	"unsafe"
)

func main() {
	x := new(int)
	y := new(int)

	ptrX := unsafe.Pointer(x)
	addressY := uintptr(unsafe.Pointer(y)) //uintr doesn't track by GC

	runtime.GC() // run garbage collector

	*(*int)(ptrX) = 100
	*(*int)(unsafe.Pointer(addressY)) = 300 //checkptr: pointer ariphmetic result points to invalid allocation
 }
```
`uintptr` - просто число. Когда `unsafe.Pointer` конвертируется в `uintptr`, GC чистит участок памяти для переменной `y`. В итоге запись в невалидный участок памяти.
Чтобы избежать этого - все операции с `uintptr` **ВСЕГДА** производятся в рамках одного выражения.
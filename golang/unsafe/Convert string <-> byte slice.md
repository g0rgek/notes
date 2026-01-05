# String to byte [slice](slice.md)
```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	var s string = "abc"
	pointer := unsafe.StringData(s)
	slice := unsafe.Slice(pointer, len(s))
	fmt.Println(slice)
}
```
unsafe.Slice возвращает [slice](slice.md) header - структуру из 3х машинных слов. Она размещается на стеке или куче, в зависимости от [Escape analysis](Escape%20analysis.md). Но compile-time [string](string.md) неизменяемые - они лежат в области text программы и не могут быть изменены.

Если попытаться изменить элемент строки (которая не была аллоцирована на стеке или хипе) по индексу - будет паника, так как строковый литерал `"abc"` лежит в READ-ONLY (TEXT) сегменте программы.
# Byte [slice](slice.md) to String
```go
package main

import (
	"testing"
	"unsafe"
)

// go test -bench=. -benchmem main_test.go

func ConvertNew(data []byte) string {
	if len(data) == 0 {
		return ""
	}

	return unsafe.String(unsafe.SliceData(data), len(data))
}

func ConvertOld(data []byte) string {
	/*
		type SliceHeader struct {
		    Data unsafe.Pointer
		    Len  int
		    Cap  int
		}

		type StringHeader struct {
		    Data unsafe.Pointer
		    Len  int
		}
	*/
	return *(*string)(unsafe.Pointer(&data))
}

var Result string

func BenchmarkConvertion(b *testing.B) {
	slice := []byte("Hello world!!!")
	for i := 0; i < b.N; i++ {
		Result = string(slice) //allocation
	}
}

func BenchmarkUnsafeConvertionNew(b *testing.B) {
	slice := []byte("Hello world!!!")
	for i := 0; i < b.N; i++ {
		Result = ConvertNew(slice) //no allocation, clearer
	}
}

func BenchmarkUnsafeConvertionOld(b *testing.B) {
	slice := []byte("Hello world!!!")
	for i := 0; i < b.N; i++ {
		Result = ConvertOld(slice) //no allocation, faster
	}
}
```
Когда присваивается значение, которое является [1.3 zero value](1.3%20zero%20value.md) для типа - включается оптимизация с использованием `memclr`. Возможно, она использует векторизацию и поэтому оно быстрее.
```go
package main

import (
	"testing"
)

// go test -bench=. clear_test.go

func BenchmarkClearWithFive(b *testing.B) {
	data := make([]int, 10*1024)
	for i := 0; i < b.N; i++ {
		for idx := range data {
			data[idx] = 5
		}
	}
}

func BenchmarkClearWithZero(b *testing.B) {
	data := make([]int, 10*1024)
	for i := 0; i < b.N; i++ {
		for idx := range data {
			data[idx] = 0
		}
	}
}
/*
BenchmarkClearWithFive-12         488779              2420 ns/op
BenchmarkClearWithZero-12         937646              1227 ns/op
*/
```


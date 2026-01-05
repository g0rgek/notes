
- В записи `aMap[key] = aMap[key] + 1`, ключ хэшируется дважды. 
  Но в случае `aMap[key]++` - только единожды.
- `aMap[key] += value` более производительно, чем`aMap[key] = aMap[key] + value`.

```go
package maps

import "testing"

var m = map[int]int{}

func Benchmark_increment(b *testing.B) { // 11.31 ns/op
	for i := 0; i < b.N; i++ {
		m[99]++
	}

}

func Benchmark_plusone(b *testing.B) { // 11.21 ns/op
	for i := 0; i < b.N; i++ {
		m[99] += 1
	}
}

func Benchmark_addition(b *testing.B) { // 16.10 ns/op
	for i := 0; i < b.N; i++ {
		m[99] = m[99] + 1
	}
}
```
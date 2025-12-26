Когда объявляется переменная без явного значения, Go автоматически присваивает zero value для этого типа.

| Type / Category                                                                                    | Zero value                                                                            |
| -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Numeric types (`int`, `uint8`, `uint`, `uintptr`, `float32`, `float64`, `complex64`, `complex128`) | `0` (or `0.0`, `0+0i`)                                                                |
| Boolean (`bool`)                                                                                   | `false`                                                                               |
| String (`string`)                                                                                  | `""` — empty string                                                                   |
| Pointer types (e.g. `*T`)                                                                          | `nil`                                                                                 |
| Function types (`func(...) ...`)                                                                   | `nil`                                                                                 |
| Interface types                                                                                    | `nil`                                                                                 |
| Slice types (`[]T`)                                                                                | `nil` — zero-length, zero-capacity, nil pointer                                       |
| Map types (`map[K]V`)                                                                              | `nil` — must initialize (e.g. `make`) before use                                      |
| Channel types (`chan T`)                                                                           | `nil` — uninitialized channel                                                         |
| Array types (`[N]T`)                                                                               | Each element is set to the zero value of `T` (so array is zero-initialized)           |
| Struct types (`struct { ... }`)                                                                    | All fields are zero-initialized (recursively) — each field gets its type’s zero value |

---
```go
package main

import "fmt"

func main() {
    var i int            // zero value: 0
    var f float64        // 0.0
    var b bool           // false
    var s string         // ""
    var p *int           // nil pointer
    var fn func()       // nil function
    var m map[string]int // nil map
    var slice []int      // nil slice
    var ch chan bool     // nil channel
    var arr [3]int       // [0, 0, 0]
    type Person struct { Name string; Age int }
    var person Person    // { Name: "", Age: 0 }

    fmt.Printf("%v %v %v %q %v %v %v %v %v %v %#v\n",
        i, f, b, s, p, fn, m, slice, ch, arr, person)
    // Output: 0 0 false "" <nil> <nil> map[] [] <nil> [0 0 0] main.Person{Name:"", Age:0}
}
```

Rune = int32 = Unicode symbol code
Руны обозначаются одинарными кавычками:
```go
package main

import "fmt"

func main() {
	var a rune = 'a'
	fmt.Printf("%d %c", a, a) // 97 a
}
```
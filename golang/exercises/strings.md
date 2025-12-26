```go
package main

import "fmt"

func main() {
	var s string = "Hello"
	for i, v := range s {
		fmt.Printf("%v, %T = %v, %T\n", s[i], s[i], v, v)
	} // --> результат выполнения ( 00, uint8, 00, int32)

	var s1 string = "Привет"
	fmt.Println(len([]rune(s1)))
}
```
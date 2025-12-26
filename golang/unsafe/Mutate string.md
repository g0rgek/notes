Важно понимать, что это возможно только если массив байт НЕ ЛЕЖИТ в READ-ONLY (TEXT) сегменте программы. Тут мы видим, что массив байт строки лежит на стеке (наверное) и строка ссылается на этот массив. Поэтому мы можем его модифицировать.
```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	data := []byte("Hello world")
	strData := unsafe.String(unsafe.SliceData(data), len(data))

	fmt.Println(strData)
	data[0] = 'W'
	fmt.Println(strData)
}
```
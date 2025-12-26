Append не модифицирует, а возвращает новый slice.

```go
func appendlnt(x []int, у int) []int {
	var z []int
	zlen := len(x) + 1
	if zlen <= cap(x) {
		// Имеется место для роста. Расширяем срез,
		z = х[:zlen]
	} else {
		// Места для роста нет. Выделяем новый массив. Увеличиваем
		// в два раза для линейной амортизированнной сложности,
		zcap := zlen
		if zcap < 2*len(x) {
			zcap = 2 * len(x)
			z = make([]int, zlen, zcap)
			сору(г, x) // Встроенная функция; см. текст раздела
		}
		z[len(x)] = у
		return z
	}
}
```

# append works on nil slices
```go
package main

import (
	"fmt"
)

func main() {
	var x []int //append will initialize an array 
	x = append(x, 0)  // [0],      len = 1, cap = 1
	x = append(x, 1)  // [0,1]     len = 2, cap = 2
	x = append(x, 2)  // [0,1,2]   len = 3, cap = 4
	y := append(x, 3) // [0,1,2,3] len = 4, cap = 4
	z := append(x, 4) // [0,1,2,4] len = 4, cap = 4
	
	fmt.Println(y, z) // [0,1,2,4] [0,1,2,4]
}
```
x, y, z ссылаются на один и тот же базовый массив. 
Дело в том, что Х имеет len = 3, те он знает только про 3 первых элемента.
Y после append присваивается слайс Х плюс новый элемент. Len увеличивается на 1. 
Х и Y содержат ссылки на один и тот же базовый массив.
Z после append присваивается слайс X плюс новый элемент. Но так как 4 элемент уже занят значением 3 (после Y), то значение 4 перезатирает значение 3.

![[Pasted image 20240601225522.png]]![[Pasted image 20240820112647.png]]
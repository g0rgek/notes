# Header
Строка - неизменяемая последовательность байт. Занимает 2 машинных слова.
```go
type StringHeader struct {
   Data uintptr
   Len  int
}
```

Неизменяемость означает, что две копии строки могут вполне безопасно разделять одну и ту же память, что делает копирование строки очень дешевой операцией.
```go
fmt.Println(s[:]) // [:] = [0:len(s)] = hello world
```
# Кодировки
## ASCII
- 0-127 значений
- 0-31 - управляющие символы
- 31-end - буквы/числа
- используется только 7 бит, старший бит не используется
Так как каждый символ - это число, то можно делать над ними арифметические операции. Мы можем узнать каким по счету идет та или иная буква:
```go
package main

import "fmt"

func main(){
 fmt.Println('w' - 'a') // 119-97
 fmt.Println('o' - 'a') // 111-97
}
```
## UNICODE
- Первые 128 символов одинаковые с ASCII
- Байты добавляются в конце
## USC-2
- В начало добавляется 2 доп. байта | FE | FF |
- Их называют **BOM (byte order mask)**
- FE - little-[Endian](../1.5%20Endians.md)
- FF - big-endian
## UTF-8
- Коды символов переменной длины от 1 до 4 байт (битовые маски)
- Первые 128 совпадают с ASCII
- Используются маски кодов вместо **BOM**
	- BigEndian - первый байт будет 0/110/1110/11110
	- LittleEndian - первый байт будет 10
## UTF-16 
- Символы кодируются 2 или 4 байтами
## UTF-32 
- Символы кодируются 4 байтами
- [rune](rune.md) тоже занимает 4 байта
# Конвертирование [rune](rune.md) и Байтов
Можно конвертировать стандартной библиотекой:
- `string <-> []byte`
- `string <-> []rune`
- `[]rune->string->byte`

Нужно писать реализацию:
- `[]byte<->[]rune`
```go
package main

import (
	"bytes"
	"unicode/utf8"
)

func Runes2Bytes(rs []rune) []byte {
	n := 0
	for _, r := range rs {
		n += utf8.RuneLen(r)
	}

	n, bs := 0, make([]byte, n)
	for _, r := range rs {
		n += utf8.EncodeRune(bs[n:], r)
	}

	return bs
}

func main() {
	s := "Hello world!!!"

	bs := []byte(s) // string -> []byte
	s = string(bs)  // []byte -> string

	rs := []rune(s) // string -> []rune
	s = string(rs)  // []rune -> string

	rs = bytes.Runes(bs) // []byte -> []rune
	bs = Runes2Bytes(rs) // []rune -> []byte
}
```
# Range over string
for range итерируется по [rune](rune.md)(!!!), а не по байтам. Так как некоторое руны занимают несколько байт, индекс может перепрыгивать некоторые значения.
```go
package main

import "fmt"

func main() {
	text := "Sr, привет 世界"
	for idx, symbol := range text { // range []rune(text)
		fmt.Printf("%d-%c ", idx, symbol)
	}
}
```

Чтобы итерироваться по байтам, нужно обращаться по индексу напрямую:
```go
text := "Sr, привет 世界"
for i := 0; i < len(text); i++ { // range []byte(text)
	fmt.Printf("%d-%c ", i, text[i])
}
```
# Узнать длину строки в рунах
```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	myStr := "hello, world🚀"
	fmt.Printf("Bytes: %d, Runes: %d\n", len(myStr), utf8.RuneCountInString(myStr))
	for i, rune := range myStr {
		
		// Print all bytes for this rune
		for _, byte := range []byte(string(rune)) {
			fmt.Printf("%d \t %v \t %T \t %v \t %v\n", i, byte, byte, rune, string(rune))
		}
	}
}
 Bytes: 16, Runes: 13
 0 104 uint8 104 h 
 1 101 uint8 101 e 
 2 108 uint8 108 l 
 3 108 uint8 108 l 
 4 111 uint8 111 o 
 5 44 uint8 44 , 
 6 32 uint8 32 
 7 119 uint8 119 w 
 8 111 uint8 111 o 
 9 114 uint8 114 r 
10 108 uint8 108 l 
11 100 uint8 100 d 
12 240 uint8 128640 🚀 
12 159 uint8 128640 🚀 
12 154 uint8 128640 🚀 
12 128 uint8 128640 🚀
```

При итерации по строке, мы получаем следующую руну [rune](rune.md). Одна руна может занимать несколько байт. Поэтому вычислять длину строки через `len(myString)` неверно, так как мы получим количество байт в строке, а не количество рун.
# Как runtime понимает сколько байт занимает символ
[Кодировка](#Кодировки) UTF-8 кодирует символы, используя от одного до четырех байтов.
Количество байт, используемых для символа, можно определить по старшим битам первого байта:
- 1-байтовая последовательность: `0xxxxxxx`
- 2-байтовая последовательность: `110xxxxx 10xxxxxx`
- 3-байтовая последовательность: `1110xxxx 10xxxxxx 10xxxxxx`
- 4-байтовая последовательность: `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx`
# Raw literal
```go
s1 := "hello\tworld" // String literal     = hello world
s2 := `hello\nworld` // RAW string literal = hello\tworld
```
# strings.Builder
Лучший способ конкатенации строк, так как он не приводит к дополнительным аллокациям.
Внутри используется [slice](slice.md) байт, в который по-умному добавляются новые строки. Используются внутренние функции рантайма, которые уменьшают количество аллокаций.
# COW-строки
- При чтении или копировании - просто копируем [header](#header) строки и читаем из общей памяти
- При записи копируем содержимое в новое место и изменяем. Получааем 2 разные строки.
- Если всего 1 ссылка на эту строку - можно изменять inplace (модифицировать байты)
[Ссылка на реализацию](https://github.com/Balun-courses/deep_go/blob/master/lessons/strings/cow_string/main.go)
# SSO
## Сравнение строк
- Сначала сравнивается длина
- Потом адреса указателей
- Потом итерация по каждому байту двух строк
## String interning
### Compiler
Go **interns** compile-time string constants, meaning identical strings share the same underlying memory:    
```go
func main() {
	const a = "hello"
	var b = "hello"
	p1 := unsafe.StringData(a)
	p2 := unsafe.StringData(b)
	fmt.Println("data pointers equal:", p1 == p2) // true
}
// `a` and `b` point to the same memory location.    
```
### Unique package
В `go 1.24` появился пакет `unique`, который позволяет делать интернирование.
```go
package main

import (
	"testing"
	"unique"
)

// go test -bench=. bench_test.go

var str1 = `Lorem ipsum dolor sit amet, consectetur adipiscing elit, 
	sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. 
	Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris 
	nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in 
	reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla 
	pariatur. Excepteur sint occaecat cupidatat non proident, sunt 
	in culpa qui officia deserunt mollit anim id est laborum`

var str2 = `Lorem ipsum dolor sit amet, consectetur adipiscing elit, 
	sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. 
	Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris 
	nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in 
	reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla 
	pariatur. Excepteur sint occaecat cupidatat non proident, sunt 
	in culpa qui officia deserunt mollit anim id est laborum`

var Result bool

// describe about:
// - space optimization
// - other types
// - synchronization

func BenchmarkConversion(b *testing.B) {
	handle1 := unique.Make(str1 + "!!!")
	handle2 := unique.Make(str2 + "!!!")

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		Result = handle1 == handle2 //compare pointers
	}
}

func BenchmarkConversionWithStr(b *testing.B) {
	sstr1 := str1 + "!!!"
	sstr2 := str2 + "!!!"
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		Result = sstr1 == sstr2 //compare byte by byte
	}
}
```
## Concatenation optimization
[Source code](https://github.com/golang/go/blob/master/src/runtime/string.go#L26)
[Ускорение конкатенации строк в Go своими руками](https://habr.com/ru/articles/417479/)
Буфер выделяется заранее, единоразовая конкатенация быстрее `fmt.Sprintf` и т.п. Буфер позволяет быстро преобразовать строчку к `[]byte` без аллоакций. Если не получилось - только тогда создается новый:

```go
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte
```

```
BenchmarkSprintfThree-10   18385520       56.60 ns/op
BenchmarkPlusThree-10      1000000000      0.2883 ns/op
```

Так же есть оптимизации для конкатенации нескольких строк разом.
```go
// src/runtime/string.go

// concatstring2 helps make the callsite smaller (compared to concatstrings),
// and we think this is currently more valuable than omitting one call in the
// chain, the same goes for concatstring{3,4,5}.
func concatstring2(buf *tmpBuf, a0, a1 string) string {
	return concatstrings(buf, []string{a0, a1})
}

func concatstring3(buf *tmpBuf, a0, a1, a2 string) string {
	return concatstrings(buf, []string{a0, a1, a2})
}

func concatstring4(buf *tmpBuf, a0, a1, a2, a3 string) string {
	return concatstrings(buf, []string{a0, a1, a2, a3})
}

func concatstring5(buf *tmpBuf, a0, a1, a2, a3, a4 string) string {
	return concatstrings(buf, []string{a0, a1, a2, a3, a4})
}
```
## Conversion optimisations
Не вызывают аллокаций преобразования:
- `string -> []byte` в рамках [for-range](slice#Range%20over%20slice)
- `[]byte -> string` как ключа [map](map.md)
- `[]byte -> string` при сравнении
- `[]byte -> string` которое используется при конкатенации строк (одно из значений объединенной строки должно являться непустой строковой константой)
```go
package main

import (
	"fmt"
	"testing"
)

func rangeWithoutAllocation() {
	var str = "world"
	for range []byte(str) { // no allocation with copy
	}
}

func workWithMapsWithoutAllocation() {
	key := []byte{'k', 'e', 'y'}

	data := map[string]string{}

	data[string(key)] = "value" // allocation with copy
	_ = data[string(key)]       // no allocation with copy
}

func comparisonAndConcatenation1() {
	var x = []byte{1023: 'x'}
	var y = []byte{1023: 'y'}

	if string(x) != string(y) { // no allocation with copy
		s := (" " + string(x) + string(y))[1:] // single alloc and copy
		_ = s
	}
}

func comparisonAndConcatenation2() {
	var x = []byte{1023: 'x'}
	var y = []byte{1023: 'y'}

	if string(x) != string(y) { // no allocation with copy
		s := string(x) + string(y) // allocation with copy
		_ = s
	}
}

func main() {
	fmt.Println(testing.AllocsPerRun(1, comparisonAndConcatenation1)) // 1
	fmt.Println(testing.AllocsPerRun(1, comparisonAndConcatenation2)) // 3
}
```
## Escape Analysis & Stack Allocation
Если [escape analysis](../memory%20model/Escape%20analysis.md) решил что переменная не покидает scope функции, то Go может про аллоцировать строку на [стеке горутины](../memory%20model/Стек%20горутины.md).
```go 
func foo() {
	s := "short" // May stay on the stack.
}
```
# Задачи
## Задача1
```go
package main

import (
	"fmt"
)

func main() {
	greet := "привет как дела" // 15 symbols (runes)
	
	fmt.Println(len(greet))    // 28 bytes
	
	fmt.Printf("%v %b %c \n", greet[1], greet[1], greet[1]) //191 10111111 ?

	var s rune = rune(greet[1])
	fmt.Println(s) //191

	runes := []rune(greet)
	fmt.Printf("%c\n", runes[1]) // р
}
```
## Задача 2
for range итерируется ПО [РУНАМ](<rune.md>):
- `i` индекс первого байта руны 
- `v` - `int32` значение из [UTF-8](https://www.utf8-chartable.de/unicode-utf8-table.pl?utf8=dec&unicodeinhtml=dec) таблицы для руны
Но одна руна может занимать несколько байт. Символ `é` занимает 2 байта. Если вывести `s[i]` то выведется числовое значение первого байта руны `é` 195. Он интерпретируется как `Ã`. 
```go
package main

import "fmt"

func main() {
	s := "héllo" // second char contains 2 bytes
	for i := range s {
		fmt.Printf("pos %d; char %c;\n", i, s[i])
	}
	
	fmt.Println(len(s)) // Кол-во байт в строке
}
/*
               
pos 0; char h; utf8 104
pos 1; char Ã; utf8 195
pos 3; char l; utf8 108
pos 4; char l; utf8 108
pos 5; char o; utf8 111
6
*/
```
Как починить? Использовать второй аргумент `v` в for range, в котором хранится код РУНЫ целиком:
```go
for i, v := range s {
	fmt.Printf("pos %d; char %c; utf %d;\n", i, v, v)
}
/*
pos 0; char h; utf 104;
pos 1; char é; utf 233;
pos 3; char l; utf 108;
pos 4; char l; utf 108;
pos 5; char o; utf 111;
*/
```


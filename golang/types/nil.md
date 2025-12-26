Является [[zero value]] для некоторых типов.
# Typed nil
## Определение
В Go значение `nil` само по себе **не имеет типа**. Но любая переменная _имеет тип_.  
Поэтому, при присваивании `nil` переменной с типом [[interface]], например:

```go
var r io.Reader = nil
```
переменная `r` получает **тип `io.Reader` и значение nil** → это и есть _typed nil_.
##  Ловушка
```go
var b []byte = nil 
fmt.Println(b == nil) // true, data=nil
```

**НО(!!!)**:
```go
var e error = (*MyError)(nil) 
fmt.Println(e == nil) // false; itab!=nil, data=nil
```
## Почему  
Потому что [[interface]] в Go устроен так:

```go
type iface struct{
	tab *itab           
	data unsafe.Pointer 
}
```

Когда присваиваеncz `(*MyError)(nil)` переменной типа [[error]]:
- `tab = *MyError`
- `data = nil`

Интерфейс не пуст → сравнение с `nil` возвращает `false`.
## Что нужно помнить
- `nil` без типа не существует внутри переменной. Переменной всегда нужен тип → это и есть typed nil.
- Интерфейс считается nil **только если и typeinfo == nil, и data == nil**.
- Частая ошибка — возвращать typed nil в `error`.
## Как избегать ловушек

Правильный способ вернуть «нет ошибки»: 
```go
return nil
```

Неправильный: 
```go
var err *MyError = nil 
return err // typed nil → превращается в non-nil error
```

# Задача 1
![[Pasted image 20251128234759.png]]
Ответ: выведется hello, так как мы никак не взаимодействуем с myTypePointer. Никакого `nil pointer dereference` не будет.
![[Pasted image 20251128234902.png]]
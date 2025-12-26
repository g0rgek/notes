context.Context - это [interface](interface.md).
Стандартных контекстов достаточно много, можно реализовывать свой.
# API
```go
func (c *Context) Deadline() (time.Time, bool)
func (c *Context) Done() <-chan struct{}
func (c *Context) Err() error
func (c *Context) Value(key any) any
```
# Отличие WithTimeout & WithDeadline
WithTimeout это обертка над WithDeadline:
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```
# Контексты с отменой
- Вызов cancelFunc (cf) отправляет в ctx.Done канал информацию.
- Вызов идемпотентен
- Вызов cancelFunc в другой функции - **антипаттерн**.
- Вызывайте cancelFunc с [defer](defer.md), чтобы автоматически отменить при выходе из функции
```go
ctx, cf := context.WithCancel(parentCtx)

go func(){
	for {
		select{
			case <-ctx.Done():
				return
			default:
				continue
		}
	}		
}()

cf() // send do ctx.Done channel
```
# Отмена дочерних контекстов
Если от контекста были унаследованы другие контексты, то при отмене родительского - все дочерние будут отменены.
Но если у родительского есть свой другой родительский, то внешний родительский отменен не будет.
```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	_, cancel = context.WithCancel(ctx)
	cancel()

	if ctx.Err() != nil {
		fmt.Println("cancelled") // wil NOT print
	}
	
}
```
# Проверка актуальности контекста
```go
package main

import "context"

func WithContextCheck(ctx context.Context, action func()) {
	if action == nil || ctx.Err() != nil {
		return
	}
	
	action()
}

func main() {
	ctx := context.Background()
	// Manually
	select {
		case <-ctx.Done():
			// cancelled
		default:
	}

	// Clever
	WithContextCheck(ctx, func(){
		//...
	})
}
```
# WithValue
В контекст можно складывать сервисную информацию, которую хотелось бы передать между стадиями:
- Айдишники
- Logger
- TraceId
```go
ctx := context.WithValue(context.Background(), "trace_id", "12-21-33") //put
id := ctx.Value("trace_id").(string) //get
```
## Перекрытие значений
Дочерние контексты наследуют все значения родительского, т.е. сначала будет поиск в начальном контексте, потом во всех родительских. Но можно и перекрыть, тогда вернется первый найденный по ключу:
```go
func main() {
	ctx := context.WithValue(context.Background(), "key", "value1")
	ctx = context.WithValue(ctx, "key", "value2")
	
	fmt.Println(ctx.Value("key").(string)) // "value2"
}
```
## Как не допустить перекрытия
Используя [определения](struct#Псевдонимы%20и%20определения), можно создать определение для нужного типа.
Так как WithValue принимает [empty interface](interface#Empty%20interface%20(any)), он сохраняет динамиеский тип и значение. И перекрытия не будет, так как типы key1 и key2 **РАЗНЫЕ**:
```go
func main() {
	type key1 string //type definition
	type key2 string //type definition
	
	const k1 key1 = "key"
	const k2 key2 = "key"
	
	ctx := context.WithValue(context.Background(), k1, "value1")
	ctx = context.WithValue(ctx, k2, "value2")
	
	fmt.Println(ctx.Value(k1).(string)) // "value1"
	fmt.Println(ctx.Value(k2).(string)) // "value2"
}
```
# Отличие TODO и Background
1. По коду: **НИЧЕМ** не отличаются (кроме метода String)
```go
type backgroundCtx struct{ emptyCtx }

func (backgroundCtx) String() string {
	return "context.Background"
}

type todoCtx struct{ emptyCtx }

func (todoCtx) String() string {
	return "context.TODO"
}
```
2. По смыслу: 
	- TODO - если не знаешь что использовать (и потом поменять)
	- Background - точно знаешь, что тут можно без других контекстов
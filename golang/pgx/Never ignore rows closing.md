## Scoped Query method
Так как rows игнорируется - они никогда не будут закрыты.
Acquire у пула будет висеть и Pool.Close() заблокируется навсегда. 
```go
if _, err := Pool.Query(ctx, sql, args...); err != nil{
	print(err)
}
```
В этом случае, **ВСЕГДА** закрывай rows:
```go
if rows, err := Pool.Query(ctx, sql, args...); err != nil{
	rows.Close()
	print(err)
}
```
## Dangling row
```go
row := Pool.QueryRow(ctx, sql, args...)
...
Pool.Close()
```
Аналогично: нужно вызвать метод Scan(), который вернет ошибку:
```go
err := Pool.QueryRow(ctx, sql, args...).Scan()
if err != nil {...}
...
Pool.Close()
```
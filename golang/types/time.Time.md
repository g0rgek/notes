Go’s time formatting unique and different than what you would do in other languages. Instead of having a conventional format to print the date, Go uses the reference date `20060102150405` which seems meaningless but actually has a reason, as it’s `1 2 3 4 5 6` in the Posix date command:
[Example](https://pkg.go.dev/time#example-Time.Format)

```go
Mon Jan 2 15:04:05 -0700 MST 2006 
0    1  2 3   4  5              6
```

In Go layout string is:

```go
Jan 2 15:04:05 2006 MST 
1   2 3   4  5   6  -7
```
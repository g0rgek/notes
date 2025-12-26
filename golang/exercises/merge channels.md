
```go
func main() {
ch1 := make(chan int, 10)
ch2 := make(chan int, 20)

ch1 <- 1
ch2 <- 2
ch2 <- 4
close(ch1)
close(ch2)

ch3 := merge[int](ch1,ch2)

for val := range ch3 {
	fmt.Println(val)
	}
}

func merge[T any](chans ...chan T) chan T {
	result := make(chan T)
	wg := sync.WaitGroup{}
	
	for _, ch := range chans {
		ch := ch
		wg.Add(1)
		go func(){
			defer wg.Done()
			for val := range ch {
				result <- val
			}
		}()
	}
	
	go func() {
		wg.Wait()
		close(result)
	}()
	
	return result
}
```
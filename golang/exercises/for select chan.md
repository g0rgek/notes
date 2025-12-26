
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	wg := sync.WaitGroup{}
	ch := make(chan string)
	mu := sync.Mutex{}

	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func(group sync.WaitGroup) {

			defer group.Done()

			mu.Lock()
			ch <- fmt.Sprintf("Goroutine result %d", i)
			mu.Unlock()
		}(wg)
	}

	for {
		select {
		case s := <-ch:
			fmt.Println(s)
		}
	}

	wg.Wait()
}

```
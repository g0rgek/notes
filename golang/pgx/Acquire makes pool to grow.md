# Question
I read pool.Stats() and it answers that pool have 1 connection and 1 acquired connection. If connection from pool is blocked in listen goroutine on WaitForNotification(), how it is possible that Exec get connection?

# Answer
When you call `pool.Acquire()`, it **creates a new connection** if all existing connections are busy, rather than strictly being limited to the max connections. The pool dynamically grows when needed (until hitting max limits).

## What's Happening in Your Code:

1. Initial State:
	* Default pool starts with 1 connection (min connections)
    - `TotalConns: 1`, `AcquiredConns: 0`  
2. When `listen()` runs:
```go
conn, err := pool.Acquire(context.Background())
```
* Acquires the 1 available connection
* Now `TotalConns: 1`, `AcquiredConns: 1` 
* This connection blocks on `WaitForNotification()`     
3. When `Exec()` is called: 
- Pool sees all connections are busy    
- **Creates a new connection** (even though initial pool size was 1)        
- Now `TotalConns: 2`, `AcquiredConns: 1` (the listen connection + new Exec connections)
## Why This Works:

1. `pgxpool` has two important settings:
    
    - `MaxConns`: Absolute limit (default is unlimited)
        
    - `MinConns`: Starting size (default is 1)
        
2. The pool will:
    
    - Start with `MinConns` (1 in your case)
        
    - Grow as needed when requests come in
        
    - Only block if it would exceed `MaxConns`
        
## The Critical Point
Your code works because `pgxpool` is designed to be **elastic** - it doesn't strictly enforce the initial pool size as a hard limit for availability. This is different from some other connection pool implementations.
# Definition

In Go, an empty struct (struct{}) is a struct type with no fields. By definition it holds no data, and thus its _width_ (size) is **zero**. For example, unsafe.Sizeof(struct{}{}) always prints 0, and even a struct composed entirely of empty structs still has size 0. 

The Go specification formalizes this: _“A struct or array type has size zero if it contains no fields (or elements, respectively) that have a size greater than zero”_. In practice, this means that struct{} values consume no storage and require no padding or alignment beyond the minimal (at least 1-byte) alignment guaranteed by the language.

Despite occupying no space, struct{} is a first-class type. You can declare variables, take addresses of them (when addressable), and use them in composite types just like any other struct. This allows idiomatic uses (see below) but also leads to surprising behaviors in memory layout and pointer comparisons. In particular, the compiler/runtime often treat all zero-sized allocations as aliases to a common “zero base” address, so multiple distinct struct{} values can end up sharing the same address. We will explore these behaviors in detail.

# Zero-size objects have &zerobased address
```go
package main

func main() {
	var a [0]int
	var b struct{}
	println(&a) //0x1400005c738
	println(&b) //0x1400005c738
}
```
# Size and Alignment of struct{}

The size of a Go type is the sum of its fields plus any padding, subject to alignment. An empty struct has no fields, so its size is zero.
For example:

```go
package main  
import "fmt"  
import "unsafe"  
  
func main() {  
    var s struct{}  
    fmt.Println(unsafe.Sizeof(s)) // 0  
}
```

Even a composite struct of empty fields has size 0, since no padding is needed when there’s nothing to align. For instance:

```go
type S struct {  
    A struct{}  
    B struct{}  
}  
var s S  
fmt.Println(unsafe.Sizeof(s)) // 0
```

This is guaranteed by the spec: if no field has non-zero size, the struct’s size is zero. Because of this, struct{} has minimal alignment (1), just like a byte. The spec’s alignment rules guarantee unsafe.Alignof(x) is at least 1 for any variable x, so an empty struct aligns to 1 byte (no extra padding).

In summary, struct{} is **zero-width, zero-sized, and consumes no storage**. This allows certain idioms where we want to express “no data” or “just a placeholder” without memory overhead.

# Arrays of struct{}

An array of empty structs also occupies no storage. By the same rule, if the element type has size 0, then the entire array’s size (length×element size) is 0. 
For example:

```go
var arr [100]struct{}  
fmt.Println(unsafe.Sizeof(arr)) // 0
```

Internally, the Go compiler/runtime does not allocate space for the elements of arr; the array exists only as a conceptual slice of length 100, but its elements overlap in the same “zero” address. In fact, **every element of an array of empty structs shares the same address**. For instance, consider:

```go
var arr [10]struct{}  
fmt.Printf("%p %p\n", &arr[0], &arr[9])  
fmt.Println(&arr[0] == &arr[9]) // true
```

On a typical run this prints a single address twice and true, because &arr[0] and &arr[9] both point to the same underlying zero-sized location. (The exact address is an implementation detail, often a special pointer in the runtime called **zerobase**. In fact, Dave Cheney showed that even two separate arrays of empty structs may have their first elements at the same address. 
For example:

```go
a := make([]struct{}, 10)  
b := make([]struct{}, 20)  
fmt.Println(&a == &b)       // false: different slice headers  
fmt.Println(&a[0] == &b[0]) // true: both back the same zero-size array
```

Here &a and &b (the slice headers) are different, but `&a[0]` and `&b[0]` compare equal because both point to the shared zero byte. In general, all elements of _all_ arrays or slices of type struct{} will effectively alias one location in memory.

# Slices of struct{}

A slice of struct{} has the usual three-word header (pointer, length, capacity) and no actual backing storage. For example:

```go
x := make([]struct{}, 1e9)  
fmt.Println(len(x), cap(x))      // prints "1000000000 1000000000"  
fmt.Println(unsafe.Sizeof(x))    // prints size of slice header (24 on 64-bit)
```

The [slice](slice.md) header size is constant (3 machine words) and its backing array is conceptually length 1e9 but occupies 0 bytes. The unsafe.Sizeof(x) will print 24 (on 64-bit Go) for the header, not counting any elements. Again, all `x[i]` refer to the same zero address:

```go
x := make([]struct{}, 5)  
fmt.Printf("%p %p %p\n", &x[0], &x[1], &x[4])  
// These will typically print the same address.  
fmt.Println(&x[0] == &x[4]) // true
```

Thus, slices of empty struct behave normally in terms of length and capacity semantics, but their elements have no distinct storage. Sub-slicing also behaves normally (length/cap change) without affecting addresses.

# Pointer Addresses and Equality

Because empty structs occupy no space, Go’s semantics around their pointers are peculiar. The language specification explicitly states that _“Two distinct zero-size variables may have the same address in memory”_. Equivalently, pointers to distinct zero-size variables _“may or may not be equal.”_ Practically, this means you **cannot rely on pointer identity to distinguish empty structs**.

For example:

```go
var a, b struct{}  
fmt.Println(&a == &b) // could be true or false, unpredictable
```

On many runs (and on the current Go compiler), this prints true because both &a and &b point to the same zero-size address[10](10.md)(https://dave.cheney.net/2014/03/25/the-empty-struct%23:~:text=Interestingly,%2520the%2520address%2520of%2520two,values%2520may%2520be%2520the%2520same). But this is not guaranteed by the spec[9](9.md)(https://stackoverflow.com/questions/67441590/avoid-same-address-for-empty-structs%23:~:text=,the%2520pointers%2520to%2520be%2520different)[3](3.md)(https://go.dev/ref/spec%23:~:text=A%2520struct%2520or%2520array%2520type,the%2520same%2520address%2520in%2520memory). In fact, Go’s compiler may optimize such comparisons: there is a known case where &a == &b for zero-sized a,b compiles to false unconditionally, even if printing their numeric addresses shows them equal[11](11.md)(https://github.com/golang/go/issues/23440%23:~:text=0x0000%252000000%2520\(empty,RET). (In other words, pointer equality for zero-size types can be arbitrarily defined and is not meaningful.)

A more subtle case involves escape analysis. If empty-struct variables do not escape to the heap, the compiler may allocate them on the stack and potentially give them different addresses (or none at all), so &a == &b might be false at runtime. But if you cause them to escape (e.g. by printing them or passing them to a function), the runtime will use a common heap address (zerobase), making &a == &b become true[12](12.md)(https://stackoverflow.com/questions/52421103/why-struct-arrays-comparing-has-different-result%23:~:text=Size%2520of%2520the%2520,a%2520mistake%2520to%2520do%2520so). In short, the result can change depending on context. The lesson is **not to compare pointers to empty structs** – there is no meaningful “identity” to observe[9](9.md)(https://stackoverflow.com/questions/67441590/avoid-same-address-for-empty-structs%23:~:text=,the%2520pointers%2520to%2520be%2520different).

One place where this behavior shows up is in maps of pointers. For example, an interviewer might ask: if you do arr := [6]*struct{}{&a} and brr := [6]*struct{}{&b}, why arr == brr can differ from &a == &b. Because the array comparison uses pointer equality semantics, it may return true if it considers the pointers equal, even if &a == &b is unpredictable. The key takeaway: **pointer equality on empty structs is unreliable by design**.

# Compiler and Runtime Handling (zerobase)

Under the hood, Go’s runtime uses a special trick to handle zero-byte allocations. In the memory allocator (runtime/malloc.go), any allocation request of size 0 immediately returns a pointer to a global variable named `runtime.zerobase`. 
For example, the pseudocode does:

```go
if size == 0 {  
    return unsafe.Pointer(&zerobase)  
}
```

This means **all zero-sized objects are given the same address** (&zerobase). (The actual address is arbitrary and hidden in the runtime, but it is constant within a program run.) As one author put it, “every zero sized object points to `runtime.zerobase`.

Because of this, _default_ allocations like new(struct{}) or taking the address of a non-escaping struct{} all return the zerobase pointer (or a stack alias of it). No new memory is used. Only when an empty struct has some data or padding (e.g. by embedding a byte or int) does allocation actually reserve distinct space.

Escape analysis in the compiler can also affect this. If a struct{} variable escapes (e.g. you take its address and store it in a heap object), the compiler may still reuse the zerobase or allocate a single heap cell for it. If it does not escape, it might simply not allocate anything at all. In any case, developers should treat pointers to empty structs as referring to a single anonymous address, not as unique objects.

# Maps with struct{} Values

A common idiom in Go is to use map[K]struct{} as a set of keys of type K. Since the value type struct{} occupies no space, this map stores only keys internally and no data for values[13](13.md)(https://stackoverflow.com/questions/22770114/any-difference-in-using-an-empty-interface-or-an-empty-struct-as-a-maps-value%23:~:text=The%2520empty%2520struct%2520and%2520empty,store%2520something%2520else%2520in%2520it). For example:

```go
set := make(map[string]struct{})  
set["foo"] = struct{}{}  
if _, ok := set["foo"]; ok {  
    fmt.Println("found foo")  
}
```

Here the struct{}{} literal means “no value” and takes zero bytes in each map entry[14](14.md)(https://stackoverflow.com/questions/22770114/any-difference-in-using-an-empty-interface-or-an-empty-struct-as-a-maps-value%23:~:text=@DanielWilliams:%2520A%2520Go%2520map%2520stores,element%2520value%2520occupies%2520zero%2520bytes). The benefit is purely semantic and a slight memory saving: unlike map[K]bool, which stores one bool per entry, map[K]struct{} stores nothing for the value. The code also clearly expresses “only keys matter”[13](13.md)(https://stackoverflow.com/questions/22770114/any-difference-in-using-an-empty-interface-or-an-empty-struct-as-a-maps-value%23:~:text=The%2520empty%2520struct%2520and%2520empty,store%2520something%2520else%2520in%2520it). (In practice, the savings are minor, but it’s a recognized optimization for very large maps or memory-sensitive code.)

Go maps themselves still have overhead for buckets, but each entry’s _value_ slot adds no payload. It’s worth noting a corner case: a map with key type struct{} can hold at most one entry, because all empty-struct keys are considered equal (there’s only one possible key value). But that is a rare use case. The more typical case is using struct{} as the value type for a key type that actually varies.

# Best Practices and Interview Notes

- **Be explicit when you need uniqueness.** Never assume different empty-struct values have distinct addresses or identities. If you _do_ need unique values (for example, to implement an enum or distinct singletons), add a dummy field. For instance, changing type Foo struct{} to type Foo struct{_ int} (or similar) ensures each allocation uses at least one word and gets its own address[15](15.md)(https://stackoverflow.com/questions/67441590/avoid-same-address-for-empty-structs%23:~:text=Replace). As one expert notes, this costs only “~1 word of memory” per object but avoids the pointer-collision pitfalls[15](15.md)(https://stackoverflow.com/questions/67441590/avoid-same-address-for-empty-structs%23:~:text=Replace).
- **Pointer comparisons are meaningless.** The Go spec allows pointer comparisons (&a == &b) only when they are guaranteed (both non-nil, same object). With zero-size variables, that guarantee is void. In interviews, if asked to compare addresses of struct{} values or their pointers, emphasize that _the result is undefined_. One StackOverflow answer bluntly states, “No. Any implementation is allowed to use the same address. You must redesign.”[9](9.md)(https://stackoverflow.com/questions/67441590/avoid-same-address-for-empty-structs%23:~:text=,the%2520pointers%2520to%2520be%2520different). In other words, don’t rely on it.
- **Use** **map[T]struct{}** **for sets** where the values are irrelevant. This idiom clearly signals “keys only” and incurs no per-value memory. It’s slightly more efficient than map[T]bool (which stores 1 byte or 1 word per entry) and often preferred for readability in Go. As noted, map[K]struct{} is a recognized idiom for sets[13](13.md)(https://stackoverflow.com/questions/22770114/any-difference-in-using-an-empty-interface-or-an-empty-struct-as-a-maps-value%23:~:text=The%2520empty%2520struct%2520and%2520empty,store%2520something%2520else%2520in%2520it).
- **Use** **chan struct{}** **for signals.** A common Go pattern is to send an empty struct over a channel when you don’t need to pass any data, only a signal. For example, quit := make(chan struct{}) and then close(quit) to broadcast a shutdown signal. Because struct{} is zero-size, this channel carries no payload and has minimal overhead[16](16.md)(https://dave.cheney.net/2014/03/25/the-empty-struct%23:~:text=While%2520this%2520article%2520concentrated%2520on,for%2520signaling%2520between%2520go%2520routines). It’s a practical use of empty structs in the standard library and idiomatic Go (e.g. in context.Context signaling[16](16.md)(https://dave.cheney.net/2014/03/25/the-empty-struct%23:~:text=While%2520this%2520article%2520concentrated%2520on,for%2520signaling%2520between%2520go%2520routines)).
- **Be careful with interfaces.** If you store pointers to struct{} in interfaces, remember that two interface values I1, I2 holding *struct{} pointers compare equal if and only if the pointers compare equal. Since pointer equality is meaningless, two interfaces holding *struct{} may be equal even if they come from separate &struct{}{} calls. (In fact, (*struct{})(unsafe.Pointer(&zerobase)) is effectively always the same pointer.) This can surprise newcomers. The rule of thumb: empty structs have only one “value” in a sense, so struct{}{} == struct{}{} is always true by value equality[17](17.md)(https://dave.cheney.net/2014/03/25/the-empty-struct%23:~:text=Why%2520is%2520this?%2520Well%2520if,They%2520are%2520in%2520effect,%2520fungible), and pointers to them are treated specially.
- **Refer to the spec.** The Go Language Specification explicitly notes these behaviors: _“Two distinct zero-size variables may have the same address in memory.”_[3](3.md)(https://go.dev/ref/spec%23:~:text=A%2520struct%2520or%2520array%2520type,the%2520same%2520address%2520in%2520memory) and _“Pointers to distinct zero-size variables may or may not be equal.”_[9](9.md)(https://stackoverflow.com/questions/67441590/avoid-same-address-for-empty-structs%23:~:text=,the%2520pointers%2520to%2520be%2520different). In an interview, quoting the spec can demonstrate understanding of why this happens. The key is that Go intentionally allows empty structs to be “fungible.”

In summary, the empty struct is a zero-sized type that Go’s compiler and runtime optimize aggressively (often by reusing a single address). This makes it useful as a lightweight placeholder (e.g. in maps and channels) but also means that any code relying on distinct storage or pointer uniqueness for struct{} is fundamentally unsafe. Experienced Go developers should know to avoid such assumptions, and interviewers often probe these subtleties. Understanding the spec guarantees and seeing examples of array and pointer behavior (as above) is essential to answering these questions correctly.
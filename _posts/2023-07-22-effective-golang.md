---
title: Effective Go programming
tags:
- Programming
- Golang
layout: post
excerpt_separator: "<!--more-->"
---

This is a collection of best practices and idioms I keep close for writing clean, concise and readable programs in [Go programming language.](https://go.dev/) <!--more-->

Jump to ...

* TOC
{:toc}

---

## 1. Initializing structures with optional parameters

Golang does not support optional function arguments, nor constructors for initializing structures. This makes initializing large structures with lot of optional paramters difficult. Commander Rob Pike came up with a good solution to this problem that is demonstrated in the snippet next.

```golang
package main

import "fmt"

type MyConfig struct {
	num int
	str string
}

// MyConfig{} will create an object with num = 0 and str = "". However,
// what if we wanted them to default to -1 and "hello" respectively,
// while also allowing callers to override selective values ?
// Keep reading.

// Implements Stringer interface for printing config later.
func (c *MyConfig) String() string {
	return fmt.Sprintf("MyConfig{num: %v, str: %v}", c.num, c.str)
}

type OptionFunc func(*MyConfig)

func WithNumber(num int) OptionFunc {
	return func(config *MyConfig) { config.num = num }
}

func WithStr(str string) OptionFunc {
	return func(config *MyConfig) { config.str = str}
}

// MyConfig's constructor function
func NewConfig(ops ...OptionFunc) *MyConfig {
	config := &MyConfig{
		// Initialize to default values
		num: -1,
		str: "hello",
	}
	for _, op := range ops {
		op(config)
	}
	return config
}

func main() {
	// Config with all defaul values
	config1 := NewConfig()
	// Config with non-default num and default str
	config2 := NewConfig(WithNumber(100))
	// Config with non-default num and str
	config3 := NewConfig(WithNumber(200), WithStr("goodbye"))

	fmt.Println(config1) // MyConfig{num: -1, str: hello}
	fmt.Println(config2) // MyConfig{num: 100, str: hello}
	fmt.Println(config3) // MyConfig{num: 200, str: goodbye}
}
```

Code is pretty self-explanatory. With a little bit of closure magic (anonymous inner function returned from `With*`), we're
able to override specific fields of the public structure while also maintaining the ability to initialize them to a sane default (if zero values don't suffice).  Note that this doesn't prevent your consumers from initializing the struct directly using `MyConfig{}`, so the recommended initialization method must be documented clearly.

Sometimes structs will have to be modified to add new fields, in which case, constructor function can be modified to initialize those
fields to non-zero default values.

## 2. Struct field ordering and memory usage
Struct packing concept may not be entirely alien to those coming from C programming language. In summary, the order in which struct fields are defined can
have a huge impact on memory, especially while processing large amounts of data modelled as structs.

Consider the below program where we have defined two structs, `MyStruct1` and `MyStruct2`, both with same fields but in different order. As long as the struct
is contained within the program (i.e is not serialized to disk or sent over a network in binary), order of fields is usually not much of a concern. However,
as the output of the program shows, `MyStruct2` take `25%` fewer bytes than `MyStruct1`, no a small difference.

```go
package main

import (
	"fmt"
	"unsafe"
)

// Note that string data type has a constant size of 16 bytes,
// since it is mainly a pointer to array of chars / runes
// in memory else where.
type MyStruct1 struct {
	x int8
	y string
	z int8
}

type MyStruct2 struct {
	x int8
	z int8
	y string
}

func main() {
	fmt.Printf(
		"Size of MyStruct1 = %v \nSize of MyStruct2 = %v",
		unsafe.Sizeof(MyStruct1{}),
		unsafe.Sizeof(MyStruct2{}),
	)
}
// Output
// Size of MyStruct1 = 32
// Size of MyStruct2 = 24
```
[Try on playground](https://goplay.tools/snippet/abpsMnOxPUU)

So what brings about this difference ? Fields are stored at [CPU word](https://en.wikipedia.org/wiki/Word_(computer_architecture)) boundaries in memory, which is usually 64 bits (8 bytes) these days. In our first struct, `x` takes only one byte, but `y` needs 16, so the compiler is forced to keep the remaining 7 bytes empty and start the string at next word boundary. Fields must align with word beginnings especially if they span across multiple words. However, when `x` and `z` are declared back to back in the second struct, compiler can pack them into a single word of 8 bytes, because they fit.

So what used to be 8 bytes for `x`, 16 for `y` and another 8 for `z` in `MyStruct1`, effectively became 8 for `x` and `z`, and 16 for `z` in `MyStruct2`, reducing the total size of the struct by 1 word or 8 bytes. Though this does reduce memory, it adds a small runtime overhead. Since two fields are packed into a single word, compiler must include instructions to "unpack" them before operating on those fields (addition, subtraction etc.), because CPU assembly instructions assume data to begin at word boundaries.

In the end, struct packing is an effective technique to reduce your program's memory consumption, though it may add a small runtime overhead. Like when doing any other performance improvements, use the data from profiling your program before and after the change to make a decision.

## 3. When are golang interfaces nil ?
Interfaces in golang have two fields, data type and value (pointer to an instance of data type). An interface is nil only when **both** these values are nil. This can be surprising to new
go devs.

```golang
package main

import "fmt"

type myError struct {}

func (e *myError) Error() string { return "Stub message"; }

func giveIncorrectError(i int) error {
	// Golang has already learned the type of interface is myError,
	// so nil checks on error returned from this method will always
	// be false
	var err *myError
	if i < 0 {
		err = &myError{}
	}
	return err
}

func giveCorrectError(i int) error {
	// No idea what kind of object this interface is pointing to
	// so nil checks will return true
	var err error
	if i < 0 {
		err = &myError{}
	}
	return err
}

func main() {
	var e1 error
	fmt.Println(e1 == nil) // Prints true

	var myErr *myError = nil
	var e2 error = myErr
	fmt.Println(e2 == nil) // Prints false

	fmt.Println(giveIncorrectError(-1) == nil) // Prints false
	fmt.Println(giveIncorrectError(1) == nil)  // Prints false
	fmt.Println(giveCorrectError(-1) == nil) // Prints false
	fmt.Println(giveCorrectError(1) == nil)  // Prints true
}
```
[Try on playground](https://goplay.tools/snippet/p54yxPbfYhA)

`Error` is by far the most used interface, and so it is considered a good practice to always return `nil` explicitly on success instead of using local variables to avoid overlooking this minor detail. It also helps to unit test both success and failure code paths, with assertions on the nil-ness of the error value returned.

## 4. Adding request IDs to slog logs
[Golang 1.21 introduced slog](https://go.dev/doc/go1.21) package into stdlib. It provided much needed improvements over `log` package that exists in golang today. As usual, the package from official go team comes with excellent documentation on how to use it. If you're using slog in microservices, one common use-case would be to tag logs with a unique request ID so all log events corresponding to a request can be quickly collected for debugging. We could do just append request ID to every log, but is repetitive and cluttering. Instead, below snippet shows how a custom log handler can help to avoid this repetition.

```golang
package main

import (
	"context"
	"os"
	"time"

	"github.com/google/uuid"
	"golang.org/x/exp/slog"
)

const CtxKeyRequestId = "RequestID"

type CustomSlogHandler struct {
	slog.Handler
}

func (h CustomSlogHandler) Handle(ctx context.Context, r slog.Record) error {
	requestId, ok := ctx.Value(CtxKeyRequestId).(string)
	if ok {
		r.AddAttrs(slog.Attr{CtxKeyRequestId, slog.StringValue(requestId)})
	}
	return h.Handler.Handle(ctx, r)
}

// package level logger
var logger = slog.New(
	CustomSlogHandler{slog.NewJSONHandler(os.Stderr, nil)},
)

func main() {
	rootCtx := context.Background()

	for i := 1; i <= 3; i += 1 {
		go func() {
			// Create a child context with request scoped values, or just requestId for now.
			requestCtx := context.WithValue(rootCtx, CtxKeyRequestId, uuid.New().String())
			logger.InfoCtx(requestCtx, "hello world")
			time.Sleep(5 * time.Millisecond)
			logger.InfoCtx(requestCtx, "goodbye world")
		}()
	}

	time.Sleep(2 * time.Second)
}

// {"time":"2009-11-10T23:00:00Z","level":"INFO","msg":"hello world","RequestID":"85186bd8-ce97-47c7-b142-cbe5f91af470"}
// {"time":"2009-11-10T23:00:00Z","level":"INFO","msg":"hello world","RequestID":"296d32fb-146b-4bac-bef0-bbefa5e0f0a2"}
// {"time":"2009-11-10T23:00:00Z","level":"INFO","msg":"hello world","RequestID":"3207aca7-ac26-4b46-a937-018c8eb7bb39"}

// {"time":"2009-11-10T23:00:00.005Z","level":"INFO","msg":"goodbye world","RequestID":"3207aca7-ac26-4b46-a937-018c8eb7bb39"}
// {"time":"2009-11-10T23:00:00.005Z","level":"INFO","msg":"goodbye world","RequestID":"85186bd8-ce97-47c7-b142-cbe5f91af470"}
// {"time":"2009-11-10T23:00:00.005Z","level":"INFO","msg":"goodbye world","RequestID":"296d32fb-146b-4bac-bef0-bbefa5e0f0a2"}
```

[Try on playground](https://goplay.tools/snippet/5dBKbzJO0VW)

Some devs prefer creating sub-loggers with all request-scoped values and saving that on to the context to be retrieve by other methods. There is nothing wrong with this, but my personal preference is to use `context` to store only data, not objects operating on that data. As usual, some benchmarking might also help break the tie if you are stuck choosing one of these approaches.

## References
1. [Self-referencial functions and the design of options - Rob Pike](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)
2. [Reddit: Logging context values using slog](https://www.reddit.com/r/golang/comments/13et07w/logging_context_values_using_slog/)
3. [Golang FAQ: Why is my nil error value not equal to nil ?](https://go.dev/doc/faq#nil_error)
4. [50 Shades of Go: Traps, Gotchas, and Common Mistakes](http://golang50shad.es/)

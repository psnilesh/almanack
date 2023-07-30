---
title: Effective Go programming
tags:
- Programming
- Golang
layout: post
---

This is a collection of best practices and idioms I keep close for writing clean, concise and readable programs in [Go programming language.](https://go.dev/)

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
	return func(config *MyConfig) {
		config.num = num
	}
}

func WithStr(str string) OptionFunc {
	return func(config *MyConfig) {
		config.str = str
	}
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
	config2 := NewConfig(
		WithNumber(100),
	)

	// Config with non-default num and str
	config3 := NewConfig(
		WithNumber(200),
		WithStr("goodbye"),
	)

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

## References
1. [Self-referencial functions and the design of options - Rob Pike](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)
---
title: Effective Go programming
tags:
- Programming
- Golang
layout: post
---

This is a collection of best practices and idioms I keep close for writing clean, concise and readable programs in [Go programming language.](https://go.dev/)

## 1. Initializing structures with optional paramters

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

## References
1. [Self-referencial functions and the design of options - Rob Pike](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)
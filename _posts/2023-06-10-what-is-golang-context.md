---
title: Context in Golang and how to use it
tags:
- Programming
- Golang
layout: post
---

`Contexts` are everywhere these days. I was first introduced to it by [AWS Go SDK v2](https://github.com/aws/aws-sdk-go-v2) and its repeated use of `context.TODO()` in example snippets. I decided to spend an afternoon trying to learn what it is, and I was glad I did. When used correctly, Context allows you to manage the lifecycle of goroutines in your program, especially ensure they don't leak and / or are terminated correctly. Lets look at an exmaple.

Imagine you've written a goroutine that writes fibonacci numbers lazily into an unbuffered channel forever, and another goroutine (e.g main) to read just N numbers from the same channel.

```golang
func fibonacci() <-chan int {
	first, second := 0, 1
	result := make(chan int)

	go func() {
		for {
			third := first + second
			fmt.Printf("Computed fib %d\n", third)
			// Since this is an unbuffered channel, this channel write will block
			// until the previous written value is read.
			result <- third
			first = second
			second = third
		}
		fmt.Printf("This goroutine will never terminate")
	}()

	return result
}

func printFibonacci(n int) {
	fibCh := fibonacci()
	for i := 0; i < n; i += 1 {
		fmt.Println(<-fibCh)
	}

	fmt.Println("Exiting from printFibonacci")
}

func main() {
	printFibonacci(5)
	time.Sleep(2 * time.Second)
	fmt.Println("Exiting from main")
}
```
[View in goplay](https://goplay.tools/snippet/offbl-japLx)

If you run the above program, you'd notice nothing odd in the output but it has a major flaw. **The goroutine thats generating the next fibonacci number is stuck on channel write call and is leaked.**  Goroutines aren't garbage collected like objects. They need to explicity terminate themselves by returning from the method. It's obvious that the infinite loop inside our fibonacci goroutine is the problem. While one leaked goroutine may seem inconsequential in such a small program, imagine leaking one from every request in a web server thats serving millions of requests a day. Keeping resources locked up more than they should will sooner or later take down your service.

In this particular instance, we could easily have passed the desired length of the sequence and exited the goroutine after as many fibonacci numbers were generated. A popular approach to prevent producers from outliving consumers is to [use another channel](https://go.dev/blog/pipelines) to tell the producer that the consumer has just had enough and that it can stop now. So there'd be two channels in total, one for sending fibonacci numbers from producer to consumer, and another for consumer to say goodbyte to producer. However, using two channels everywhere can be inconvenient and error prone. It also soon becomes complicated when we want do things like cancel all child goroutines of a parent goroutine. Fortunately, some one at Google thought the same and wrote a package for passing around cancellation signals across goroutines that was incorporated into Golang's standard library. That package was `Context`.

The idea behind context's cancellation implementation is very similar to the two channel approach mentioned above. Additionally, it keeps track of parent-child relationship of goroutines (with a little bit of our help) and automaticallly cancells all child goroutines when a parent is going away. This is an important feature. To undestand why, imagine you have webserver thats launching a goroutine for every request. This goroutine then launches mutliple other goroutines for reading / writing to database, making API calls to other services etc.  It doesn't make sense for any of this secondary goroutines to outlive the original request scoped goroutine. Context and cancellation can help us achieve just that.

Lets incorporate Context into earlier program to ensure our goroutine terminates this time.


```golang
func fibonacci(ctx context.Context) <-chan int {
	result := make(chan int)

	go func() {
		defer close(result)
		first, second := 0, 1
		terminate := false
		for !terminate {
			// A quick example on select statement https://go.dev/tour/concurrency/5
			select {
			case <-ctx.Done():
				// ctx.Done() will receive a message when the parent wants this
				// coroutine to shutdown.
				terminate = true
			case result <- (first + second):
				third := first + second
				first = second
				second = third
			}
		}
		fmt.Println("Terminating fibonacci generator goroutine")
	}()

	return result
}

func printFibonacci(n int) {
	// Background is the base context that stays alive as long as the program.
	// Here, we create a new context to pass to our fibonacci goroutine.
	ctx, cancel := context.WithCancel(context.Background())
	// Schedule cancellation invocation at the end of this method. This will
	// send a signal on ctx.Done() channel. Receivers of ctx are expected to
	// wait for this signal and exit from their goroutine handler as soon as
	// possible.
	defer cancel()
	fibCh := fibonacci(ctx)
	for i := 0; i < n; i += 1 {
		fmt.Println(<-fibCh)
	}

	fmt.Println("Exiting from printFibonacci")
}

func main() {
	printFibonacci(5)
	time.Sleep(2 * time.Second)
	fmt.Println("Exiting from main")
}
```
[View in goplay](https://goplay.tools/snippet/9JgYRSjMoBl)

The advantage of a context may not be immediately apparent in this small example, but we've accomplished cancellation of our infinite fibonacci generator coroutine by doing two things, 1) creating a context and scheduling its cancellation at the end of `printFibonacci` method, and 2) listening on `ctx.Done()` channel from the producer and exiting if any message was found. You can repeat this pattern from `fibonacci(..)` method if you wanted to, in which case all the goroutines you launch there would also be recursively cancelled when the control exits `printFibonacci` method.

Its important to note that goroutine cancellation is co-operative. That's to say, parent cannot force child goroutine to exit if it isn't listening on `ctx.Done()` properly. If you're writing a method that's accepting `context.Context` as the first parameter and launching coroutines from within, its up to you to ensure all the goroutines you launched are properly terminated upon receiving a termination signal from the parent context.

This pattern has become so popular, and can be found in Go standard library and of course, lot of popular third party libraries. By convention, methods that launch goroutines accept `context.Context` as the first argument so the caller can manage their lifecycle safetly. Someimtes, we just want to invoke a method without caring much about goroutines and their lifetimes. That's where `context.TODO()` comes in.  According to the [documentation](https://pkg.go.dev/context#TODO),

> TODO returns a non-nil, empty Context. Code should use context.TODO when it's unclear which Context to use or it is not yet available (because the surrounding function has not yet been extended to accept a Context parameter).


Besides using to manage goroutine lifetimes, `Context` can also be used to store request scoped values while writing web sevices (roughly equivalent to `ThreadLocal` in thread-per-request java services). They can also be used to automatically schedule cancellation of goroutines (and all their children) if they don't finish in a specified duration. They're all very easy to understand and use, and are documented well in the links given below.


## References
* [Go Concurrency Patterns: Context](https://go.dev/blog/context)
* [Context package documentation](https://pkg.go.dev/context)
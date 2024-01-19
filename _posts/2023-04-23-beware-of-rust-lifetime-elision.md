---
title: Rust lifetimes and Elision
tags:
- Programming
- Rust
layout: post
excerpt_separator: "<!--more-->"
---

# Introduction to Lifetimes

Much has been said and written already about Rust's lifetimes. In summary, they enable the Rust borrow checker to determine how long a reference will be valid for. <!--more--> This is crucial because operating on a dead or stale reference can result in unpredictable behavior and memory unsafe programs, going back on the promise of Rust.

Lets use a simple example to demonstrate how lifetimes help the borrow checker do its job.

```rust
fn get_last_word(text: &str, seperators: &str) -> &str {
    // Skipping logic for brevity. Assume the last word begins at
    // index i and ends at j + 1
    &text[i..j]
}

fn main() {
    let text = "Hello world. How are you ?";
    let separators = " .?";
    let last_word = get_last_word(text, separators);
    assert_eq!("you", last_word);
}
```

Abvoe method accepts two string references, a text and string of seperator characters and returns a reference to the last word from the text. However, this will fail to compile with below error,
<pre>
  |
2 | fn get_last_word(text: &str, seperators: &str) -> &str {
  |                        ----              ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the
    signature does not say whether it is borrowed from `text` or `seperators`
    help: consider introducing a named lifetime parameter
  |
2 | fn get_last_word<'a>(text: &'a str, seperators: &'a str) -> &'a str {
  |                 ++++        ++                   ++          ++

</pre>

Rust compiler is giving us some very helpful hints at what the problem is. Compiler cannot infer how long the reference returned from `get_last_word` is valid for. Since we're returning a slice of `text`, the lifetime of reference is tied to that of the `text` and they need annotated with same lifetimes. Lets try that, with a small change to our main function as well.

```rust
fn get_last_word<'a>(text: &'a str, seperators: &'a str) -> &'a str {
    // Skipping logic for brevity. Assume the last word begins at
    // index i and ends at j + 1
    &text[i..j]
}

fn main() {
    let text = "Hello world. How are you ?";
    let last_word;
    {
        // error[E0597]: `separators` does not live long enough
        let separators = String::from(" .?");
        last_word = get_last_word(text, separators);
        // separators cease to exist at the end of this scope
    }

    println!("Last world is {}", last_word);
}
```
Unfortunately, this doesn't compile either. Here, `separators` don't live as long as `text` so it clearly has a shorter lifetime. They can't be used to invoke `get_last_word` because the method requires lifetimes of both arguments to match.

Since the lifetime of output reference depends only on the lifetime of `text`, the correct resolution here is to annotate `seperators` with a lifetime of its own, `'b`, like this,
```rust
fn get_last_word<'a, 'b>(text: &'a str, seperators: &'b str) -> &'a str {
    // Skipping logic for brevity. Assume the last word begins at
    // index i and ends at j + 1
    &text[i..j]
}
```

It should be obvious why this works. The `separators` string could be dropped immediately after the method returns for all we care without violating any memory safety guarantees (no dangling pointers etc). One burning question I had when I started learning lifetimes was that why couldn't the compiler infer it by itself and this clarified why. Compiler could have added those lifetimes insteading of asking us to do it, but that wouldn't have resulted in the behavior we wanted. **Compiler can understand only what you've written, it cannot infer what you intended to happen.** Taking actions on behalf of programmer hence might result in violation of [principle of least astonishment](https://en.wikipedia.org/wiki/Principle_of_least_astonishment).


# Lifetimes and memory safety

Lifetimes help ensure memory safety in more ways than is obvious at first. Consider below snippet that is building on top of our previous examples,

```rust
// get_last_word(..) definition is same as before.
fn main() {
    let mut text = String::from("Rust is memory safe, ");
    let last_word = get_last_word(&text, ",. ");
    // Lets modify the text after the fetching the last word.
    text.push_str(" but you already knew that.");
    // Print the last world
    println!("{}", last_word);
}
```
This program will fail to compile with below error,

<pre>
error[E0502]: cannot borrow `text` as mutable because it is also borrowed as immutable
  --> src/main.rs:10:5
   |
9  |     let last_word = get_last_word(&text, ",. ");
   |                                   ----- immutable borrow occurs here
10 |     text.push_str(" but you already knew that.");
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
11 |     println!("{}", last_word);
   |                    --------- immutable borrow later used here

</pre>
The error must be obvious to those familiar with Rust's ownership rules. `push_str` is mutably borrowing `text` while `last_word` has already immutably borrowed it.  According to Rust's ownership rules, you can either have N immutable references or a single mutable reference to an object, but not both. That is the text book explanation at least. So what do we do ?

Over time, I've begun to see these errors not as borrow checker giving me a fit, but as potential memory corruptions that are inherent in the program. Lets consider for a moment what we're trying to do here. We've obtained a reference (`last_world`) to a slice of a string and then we're mutating the original string. The validity of the `last_word` after depends on the sort of changes made to the source string while it was mutably borrowed. What if we hard cleared the string, truncated it, or removed the last word just for the heck of it ? Well, `last_word` would no longer be a valid reference and using it would be a bad idea. Here's the kicker, **the compiler cannot infer which mutations to `text` will render your reference invalid, so it makes a blanket assumption that any mutation will, and refuse to compile.** This might sound like overly restrictive at first, and it is somewhat so, but it is essential to enforcing Rust's memory safety guarantees.

Another way to think of it is this :- when we calculated `last_word`, `text` had an internal state `x`. Any mutable borrow, like `push_str` could potentially change that state to, say `y`. Is it safe to assume a reference that was created for state `x` is valid for `y` ? Rust says no.

How do we solve it ? It really depends on what you're trying to do. In the above example, I could just move the `println!` before the `push_str` call and it would compile successfully. If I wanted `last_word` after `push_str` call, I could clone it as well so it isn't tied to `text` anymore.


# Lifetime Elision

To make programmers' life easier, Rust development team came up with a certain set of rules where the compiler would attempt to infer lifetimes automatically instead of forcing the programmer to declare them. There are four of them, and they are well documented with loads of examples in [Rust docs](https://doc.rust-lang.org/nomicon/lifetime-elision.html) that it's not worth repeating here.

# When Elision goes wrong

The point I want to emphasize however is that elision is simply a _convenience mechanism_ that most of the time, does what the programmer would have done themselves. When it doesn't work, we'd be in for a surprise and a wild debugging ride. I'll try and demonstrate this by implementing a simple stack,

```rust
struct Stack<'a> {
    vec: Vec<&'a str>
}

impl<'a> Stack<'a> {

    fn new() -> Self {
        Stack {
            vec: vec![]
        }
    }

    fn push(&mut self, top: &'a str) {
        self.vec.push(top);
    }

    fn pop(&mut self) -> Option<&str> {
        self.vec.pop()
    }
}

fn main() {
    let mut stack = Stack::new();
    stack.push("hello");
    stack.push("world");

    let world = stack.pop();
    let hello = stack.pop();

    println!("{} {}", hello.unwrap(), world.unwrap());
}
```

There's some new use of lifetimes here, especially in structs, that we haven't discussed before. Just know that they serve the same purpose as they do in functions, if not more, i.e tie the lifetime of those references to that of struct objects.

Above program will however fail to compile with below error,

<pre>
error[E0499]: cannot borrow `stack` as mutable more than once at a time
  --> src/main.rs:28:17
   |
27 |     let hello = stack.pop();
   |                 ----------- first mutable borrow occurs here
28 |     let world = stack.pop();
   |                 ^^^^^^^^^^^ second mutable borrow occurs here
29 |
30 |     println!("{} {}", hello.unwrap(), world.unwrap());
   |                       ----- first borrow later used here

</pre>

Why is `stack` being borrowed more than once at the same time ? We are only making two `pop` calls back to back after all. The culprit, though not obvious at first, is lifetime elision, specifically in the `pop` method, because it desugars to,

```rust
    fn pop(&'b mut self) -> Option<&'b str> {
        self.vec.pop()
    }
```
Lifetime elision added a unique lifetime to references in that method that implies the output reference is now tied to the lifetime of mutable borrow. We're keeping the mutable borrow alive by keeping the popped reference alive (i.e printing it later in the program). The second pop hence cannot mutate the stack again because that could invalidate the first popped reference that is being used later in the program, and that could violate memory safety guarantees.

While it may seem like we've hit a wall at first, the solution here is rather simple. All the references in the stack have a lifetime of `'a` but elision decided that of popped reference should be `'b` because it didn't know any better. The program can be fixed by changing the output of `pop` to `Option<&'a str>`. This tells the compiler that popped reference is valid as long as the `Stack` struct is, irrespective of any future mutations.

Its well worth noting that Rust refused to compile our program with incorrect lifetimes, unlike other languages that might have compiled it and failed at runtime in similar situatations. The benefits of elision always outweighs its cons, but this "limitation" helped me gain better understanding and mental model of Rust lifetimes in general.

# Conclusion
Rust programs and compiler errors makes a lot more sense if we start to think about the memory safety of our code. Lifetime and elisions are merely tools and mechanisms that are used to express our intentions to the compiler so that it can determine how safe our programs actually are. However, this may not always go according to the plan, as we've found out. Even so, lifetime elision has made Rust a better language by avoiding a lot of biolerplate code and making some simple programs easy for beginners.


Credit: This article is just rubber ducking one of the quirks related to lifetimes that I read in another fantastic blog, [Common Rust Lifetime Misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md). Every Rust programmer should read it at least once before they get into serious Rust development.
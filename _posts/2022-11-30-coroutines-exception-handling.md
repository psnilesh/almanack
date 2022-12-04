---
title: Exception handling in Kotlin Coroutines
tags:
- Programming
layout: post
---
Coroutines are, conceptually, light weight threads that has changed how developers write concurrent programs in Kotlin. They are built using suspendable computations, functions that can _suspend_ its execution in one thread, and resume later from where it left off potentially in a  different thread. `suspend` functions are also [syntatic sugar over callbacks, as explained by Roman Elizarov](https://www.youtube.com/watch?v=_hfBv0a09Jc). It enabled developers to write concurrent programs in an imperative / procedural style. Moroever, co-operative coroutine scheduler ensures efficient use of native threads by scheduling multiple coroutines on a single thread so when one coroutine is blocked on, say a network request, the other coroutine can get some work done instead of simply blocking or keeping the thread idle. When properly applied, this can significally increase the throughput of web services for certain kind of workloads. 


While coroutines are easy use, handling exceptional cases get tricky real quick. Instead of explaining a boat load of theory about how exceptions are handled, I'm going to provide here few examples that has helped me model how coroutines behave when exceptions are left unhandled.

### Point 1) Suspend functions can throw and handle exceptions like regular functions
Suspend functions running in same coroutine behave just like regular functions when it comes to propagating exceptions. Exceptions can be thrown, caught and re-thrown as you would in a coroutine-less program. 
<pre>
<code class='kotlin-playground'>
import kotlinx.coroutines.*

fun main() = runBlocking {
    try {
        taskOne()
        taskTwo()
    } catch(e: IllegalStateException) {
        println("Exception caught: $e")
    }
}

suspend fun taskOne() {
    delay(1000)
    error("taskOne failure")
}

suspend fun taskTwo() = println("Never called.")
</code>
</pre>
Even when functions suspend in one thread and resume in another, coroutines will take care of carrying over exceptions so we can write traditional looking procedural programs with conventional exception handling. 


### Point 2) Top level _launch_ and _async_ coroutines handle exceptions differently
`launch` and `async` are builders used for creating new coroutines. The former is used for fire and forget blocks and latter
when the coroutine has something to return.  When exceptions are thrown from `launch` coroutines, its execution is cancelled and error is propagated to the parent, who in turn will cancel all its children and propagate the error to its parent and so on until the control reaches root coroutine. We can usually set up an `UncaughtExceptionHandler` to catch such exceptions. Async however, will hold on exceptions until we call `await()` on `Deferred` instance if it is a top level coroutine. Otherwise, it propagates up the error just like `launch`.

<pre>
<code class='kotlin-playground'>
import kotlinx.coroutines.*

fun main() = runBlocking {
    val scope = CoroutineScope(Job())
    val job = scope.launch {
        delay(1000)
        error("Launch error")
    }
    // join will not rethrow above exception, and you'll see the output of below print statement.
    // Coroutines launched using `launch` builder does not allow users to handle failures 
    // for good structured concurrency, and will always propagate the error to parent kicking
    // off a chain cancellation. The default UncaughtExceptionHandler will in this case log the
    // exception to console.
    job.join()
    println("Main exited successfully")
}
</code>
</pre>
On the other hand, top level `async` coroutines will not re-throw exception until and unless we call `await` method of the deferred
object.
<pre>
<code class='kotlin-playground'>
import kotlinx.coroutines.*

fun main() = runBlocking {
    val scope = CoroutineScope(Job())
    val deferred = scope.async {
        error("Launch error")
    }
    delay(1000)
    // Using join will not rethrow the exception but simply wait for coroutine to complete.
    deferred.join() 
    println("Completed join successfully.")
    // Await will either return the result or throw the exception coroutine failed with
    deferred.await() 
    // Below line will not be printed.
    println("Main exited successfully")
}
</code>
</pre>

### Point 3) Non-root _async_ coroutines propagate exceptions immediately
`Async` coroutines will hold exceptions inside `Deferred` objects only if they are top level coroutines, i.e coroutines
started directly from the scope. Nested coroutines will propagate exceptions to parent without
waiting for the user to invoke `await()` method on its deferred object. This is necessary to guarantee consistent stuctured concurrency behavior. 

<pre>
<code class="kotlin-playground">
import kotlinx.coroutines.*

fun main(): Unit = runBlocking {
    var scope = CoroutineScope(Job())
    val job = scope.launch {
        async {
            error("launch -> async error")
        }
    }

    // Job will fail (but won't rethrow the exception here since we used launch) and log
    // the above exception to console.
    job.join()


    scope = CoroutineScope(Job())
    val deferred = scope.async {
        // Exception thrown below is not held inside the Deferred object
        // but propagated to parent immediately. The behavior is same
        // with or without calling await().
        // Try catch construct cannot detect such errors, as demonstrated
        // below.
        try {
            async {
            	error("async -> async error")
        	}
        } catch(e: IllegalStateException) {
            println("This catch will not be called.")
        }
    }
    try { 
	    deferred.await()   
    } catch (e: Exception) {
        println("Caught exception: $e")
    }
}
</code>
</pre>

To handle such async exceptions, wrap `async` coroutines in a `coroutineScope`,
<pre>
<code class='kotlin-playground'>
import kotlinx.coroutines.*

fun main(): Unit = runBlocking {
    val job = async {
        try {
            coroutineScope {
                async { error("async error") }
            }   
        } catch(e: IllegalStateException) {
            println("Caught async exeption: $e")
            println("Returning default value ..")
            42
        }
    }
    // await() doesn't throw exception because it was already handled 
    // by the parent coroutine.
    println("Job ouput = " + job.await())
}
</code>
</pre>

### Point 4) An unhandled exception thrown from a coroutine will cancel its parent and all siblings

This forms the basic premise of strucutured concurrency. The concept is not unlike a local variable falling out of scope (or destroyed) when the control exits the scope it was declared in.  Consider the below program where we start three coroutines, deliberately fail one, and observe as its sibling and parents are automatically cancelled.

<pre>
<code class='kotlin-playground'>
import kotlinx.coroutines.*

fun main(): Unit = runBlocking {
    var scope = CoroutineScope(Job())
   	scope.launch {
        val job1 = launch {

            launch {
                delay(1000)
                error("Fatal error")
            }
            
            try {
                delay(10000)
            } catch(e: CancellationException) {
                println("job1 cancelled")
                // Always rethrow CancellationException for the sake of
                // your program's correctness.  Coroutine library
                // is counting on it.
                throw e
            }
        }
        
        val job2 = launch {
            try {
                delay(10000)
            } catch(e: CancellationException) {
                println("job2 cancelled")
                throw e
            }
        }
        
        job1.join()
        job2.join()
    }.join()
    println("Exiting main normally..")
}
</code>
</pre>

Above program will show that siblings job1 and job2 was cancelled when a child coroutine failed to handle an exception (or in this case, deliberately threw one). 

### Point 5) Use a Supervisor job to prevent auto cacellation of coroutine parent and siblings
So far we've seen coroutines cancelling everything when an Exception is left unhandled. There are use-cases where this isn't acceptable. To prevent a coroutine from propagating its error to parent and thus triggering a chain cancellation, create coroutine scopes with `SupervisorJob`. Coroutines launched as the **immediate** child of `SupervisorJob` do not propagate exceptions to their parents. In fact, none of their ancestors or siblings are even aware that they have failed. This is demonstrated by the program below.

<pre>
<code class='kotlin-playground'>
import kotlinx.coroutines.*

fun main(): Unit = runBlocking {
    var scope = CoroutineScope(SupervisorJob())

    // Job1 failure does not cancel job2 or job3.
   	val job1 = scope.launch {
        launch {
            delay(1000)
            error("Fatal error")
    	}

        launch {
            try {
                delay(2000)
            } catch (e: CancellationException) {
                println("Cancelled because SupervisorJob influence " + 
                       " only directly launched or first level coroutines.")
            }
        }
    }
    
    val job2 = scope.launch {
        try {
            delay(2000)
        } catch(e: CancellationException) {
            println("Never printed : $e")
        }
    }
    
    val job3 = scope.async {
        delay(2000)
        42
    }
    println("Result is ${job3.await()}")
    job1.join()
    job2.join()
}
</code>
</pre>

Note that this cancellation behavior is applicable only to coroutines that are launched directly from scope initialized with `SupervisorJob`. Any nested corourines will behave as usual (i.e recursively cancel everything). There is also a coroutine builder
`supervisorScope` that does the same thing,

<pre>
<code class='kotlin-playground'>
import kotlinx.coroutines.*


fun main(): Unit = runBlocking {
    supervisorScope {
        launch {
            delay(1000)
            error("launch error")
        }.join()
        
        val answer = async {
            delay(2000)
            42
        }.await()
        println("Printing $answer to show I'm not cancelled yet ! ")
    }
    println("Completed supervisorScope without exception..")
    
    try {
        // In contrast, coroutineScope will throw the exception it failed with...
        coroutineScope {
            launch {
                delay(1000)
                error("launch error")
            }.join()

            val answer = async {
                delay(2000)
                42
            }.await()
            println("Answer = $answer")
        }
    } catch (e: Exception) {
        println("coroutineScope threw exception: $e")
    }
    
    println("Main exiting successfully...")
}
</code>
</pre>


In short, there are some weird spots when combining coroutines and exceptions. Its pretty straight forward when exceptions are thrown between suspend functions within a single coroutine but inter-coroutine exceptions must be carefully designed and tested. Classes like [Result](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/) can help pass exceptions between coroutines but without actually throwing them. 

There's obviously more to the story than whats written here. Some articles I found immensely helpful are linked below,
* [Kotlin official documentation](https://kotlinlang.org/docs/exception-handling.html)
* [Medium - Exceptions in coroutines](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)
* [Lukas Lechner - Why exception handling with Kotlin Coroutines is so hard](https://www.lukaslechner.com/why-exception-handling-with-kotlin-coroutines-is-so-hard-and-how-to-successfully-master-it/)
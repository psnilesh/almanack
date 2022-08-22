---
title: Designing a large scale notification scheduler
tags:
- Microservices
excerpt_separator: "<!--more-->"
layout: post
---

I was recently presented a problem of designing a scalable service for scheduling notifications / callbacks that can service large no. of clients simultaneously. While scheduling a task in a single machine is trivial (`Cron`), doing so in a distributed environment is easier said than done. This post is a summary of my thought process leading up to a viable solution, concluding with some thoughts on potential improvements. <!--more-->

## Requirements
Design a REST API service that allow its clients to schedule a callback (HTTP GET) as far out as 7 days. Input to `Schedule` request will contain a fully qualified http(s) URL and a UTC timestamp at which the endpoint is to be called.
```
ScheduleCallbackRequest {
    callbackUrl: String, 
    callbackAt: UTCTimestamp
}
```
Since the purpose of this exercise is to think through challenges associated with building something like this, we will refrain from using any *scheduling service* offered by cloud platforms (e.g AWS Cloudwatch events).

## Technical specifications
Lets start by dissecting the problem into technical requirements :-
* Our service must be able to accept schedule requests from any authenticated (and authorized) client at all times. We'll stick with scheduling for now and leave authentication out of our discussion. There are abundant solutions for service to service AuthN / AuthZ ranging from AWS IAM to OAuth2.0 - just use any one of them. 
* Wherever time is mentioned, granularity will be limited to seconds. Systems that want to schedule tasks with sub-second accuracy are better off doing so themsevles locally. 
* A callback at precisely the requested time (`callbackAt`) is near impossible due to network delays, clock skews and other latent behaviors in design. However, our sytem should guarantee a grace period SLA of say, 30 seconds, within which time the callback must be made.
* We'll adhere to at-least once delivery mechanism, ensuring every callback is invoked one or more times. 
* To have a sense of the scale of solution we need to build, lets assume this service will cater to **hundreds of clients scheduling thousands of callbacks per second**.

## Approach 1: A potato solution using coroutines
The first solution that ocurred to me was naive, incomplete and had a severe flaw. Nevertheless, I'll still explain it quickly here. 
![](https://miro.medium.com/max/570/1*DFxGm7k7q3PvecP15nn1Fw.jpeg)
*Fig 1. Fleet of EC2 instances fronted by Elastic Load balancer*

As shown in the above architecture diagram, there isn't a lot of moving parts in it. It was a standard service whose compute was distributed across AZs and fronted by a  load balancer.  The bulk of heavy lifting was done within the compute instances, through the magic of [Coroutines](https://kotlinlang.org/docs/coroutines-overview.html). If you're unfamiliar, couroutines are conceptually light weight threads that can run several asynchronous tasks that aren't compute intensive on the same thread. Recent hype around coroutines was a result of Android adopting Kotlin as a first class language, which supported coroutines quite well. What appealed to me most was that we could launch [tens of thousands of coroutines](https://kotlinlang.org/docs/coroutines-basics.html#coroutines-are-light-weight) in the same process / instance that shared a limited thread pool without exhausting any sytem resources. It was the kind of scale our solution required.

API handler might have looked like something like this,

```kotlin
class ScheduleCallbackController {

    fun handle(request: ScheduleCallbackRequest): ScheduleCallbackResponse {
        // Validate request (expired timestamp, invalid url etc.)
        val delayMs = milliSecondsUntil(request.callbackAt)
        coroutineScope.launch {
            delay(delayMs)
            httpGet(request.callbackUrl)
        }
        // launch() is non-blocking and would return immediately after
        // job was submitted, allowing us to return an aknowledgement
        // to our clients.
        return ScheduleCallbackResponse(OK)
    }

}
```

As mentioned earlier, this solution suffered from a fatal flaw. Application state, i.e callbacks pending, was tied to particular instances and would be lost
when they were taken down. Since coroutines are ephermal, even something as trivial as a new deployment would result in losing all scheduled callbacks
from that instance when the JVM process was restarted. This was simply not acceptable. My eagerness to use the latest weapon from my arsenal was replaced by lingering anguish. Though I thought of separating state into an external store, I couldn't help but feel I was hacking around something severe I overlooked. So I started over.

## Approach 2: SQS and message visibility
The previous approach wasn't a complete waste of time, since it made me realize the state of program had to be maintained elsewhere. The requirement however didn't explicitly call for a persistent store, since the requests are technically kept around only until they expire. A queueing model felt more appropriate. AWS SQS is a fully managed queueing service that can send, store (14 days), and receive messages at any volume, without losing messages or requiring other services to be available. Between Standard and FIFO SQS queues, former seemed more appropriate for the problem's functional and scale requirements. 

Now the callback requests are qeueud, they had to be retrieved and processed around the time they expire. [SQS SendMessage API](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessage.html) takes in a parameter **DelaySeconds** that's defined as

>  The length of time, in seconds, for which to delay a specific message. Valid values: 0 to 900. Maximum: 15 minutes. Messages with a positive DelaySeconds value become available for processing after the delay period is finished. If you don't specify a value, the default value for the queue applies.

So callback requests that are coming due 15 minutes, can be pushed to the SQS queue with DelaySeconds equal to time until expiry. Requests that need longer wait periods, can be pushed to the qeueue with 15 mins of DelaySeconds, and repeat the process until we are closer to expiring. 

Consider a sample request whose callback time is 50 minutes in the future, 
1. API handler will push the request to SQS queue with DelaySeconds 0.
2. A compute (lamda, EC2 etc) polling the queue will receive the message within few seconds, calculate there is 50 more minutes to go before invoking callback,
   and so enqueue it back to the queue with a delay period of 15 minutes.
3. After 15 minutes (more or less), compute will once again receive the same message. Since there is still 35 minutes to go, it once again push the message
   back to the queue with a delay period of 15 minutes. When the message is received for a third time, delay period is set to 5 minutes. 
4. Fourth time the message is received, compute instance will realize its time for callback by comparing the `callbackAt` timestamp and current timestamp. It proceeds to issue one, bringing the life of the request to an end. 

Below sequence diagram summarizing interactions within various components in our system,

![](https://www.planttext.com/api/plantuml/png/dP51ImGn38Nl_HLXBtPWXVNWmG5bbps81s55_07TcOTJJDkrIHN_lHtNWO71YvT0IDxt7ibMr6KjWOqhcc89HsIpPu-eT7b7pzs0lZ3oxl3GalnsUyTyTDqRsP9vJUe36ZDV7QLF1GKjdOeCP3FU2qJNr8FT5ztIHXhpit58N3KpGKO7_u57YBXNY6sCOwLLDtds2H8lbBeKG7q1-KXNrnHybDKVo373CiBDPWm15ipe8rKcBDSCf8Fxfy44tTLJKaoVjbdO-RDN7IxGvoUqDAYUxg5sqhnahfZOcsqjzN7V)
*Fig 2. Sequence diagram ([source](https://www.planttext.com/?text=dP51ImGn38Nl_HLXBtPWXVNWmG5bbps81s55_07TcOTJJDkrIHN_lHtNWO71YvT0IDxt7ibMr6KjWOqhcc89HsIpPu-eT7b7pzs0lZ3oxl3GalnsUyTyTDqRsP9vJUe36ZDV7QLF1GKjdOeCP3FU2qJNr8FT5ztIHXhpit58N3KpGKO7_u57YBXNY6sCOwLLDtds2H8lbBeKG7q1-KXNrnHybDKVo373CiBDPWm15ipe8rKcBDSCf8Fxfy44tTLJKaoVjbdO-RDN7IxGvoUqDAYUxg5sqhnahfZOcsqjzN7V))*

### Scaling and SLA management
How well does this solution scale ? Per AWS, Standard queues support a nearly unlimited number of API calls per second, per API action (SendMessage, ReceiveMessage, or DeleteMessage). However, there can only be a maximum of approximately 120,000 in flight messages which might become a problem since we're effectively reading back requests at roughly every 15 minutes. If it does become an issue, either request a quote increase or distribute messages across multiple queues so no single queue will have more than 120,000 in flight messages. These details can be figured out by working backwards from the throughput expected by clients.

As for the compute platform for queue pollers, there are both classic and serverless options to chose from. Both have its own advantages and shortcomings. If a lambda is event sourced directly by SQS, it *might* hit concurrency limits when queue is backed up with millions of requests. A static fleet of hosts would seem more appropriate to counter this, but isn't as elastic as lambda to queue load. Hence, to meet client SLAs, the fleet will likley have to be kept scaled
high enough at all times to handle max requests from all clients simultaneously.


### Failure and risk management
> *"Everything fails, all the time"* - Werner Vogels, AWS CTO

No design would be complete without touching upon failure handling and risks. 

* **Missing customer SLA** - Undisputedly, the biggest risk. As mentioned earlier, the callback should be made within 30 seconds of expiry, but unfortunately this isn't something that can be enforced in SQS. The only way to ensure the service can honor the SLA at peak load is to rigorously and repeatedly load test. 

* **Bad deployments** - Any problems with application code should be rolled back automatically so as to affect minimal number of requests. Requests that failed
would have been moved to a dead letter queue. They can be replayed if not expired, dropped or dispatched otherwise per client expectations.

* **Multiple callbacks** - Standard SQS queues follow at least once delivery model, so certain messages can be expected to be received more than once. This could potentially trigger multiple callbacks when they expire. Exactly once delivery model is incredible difficult to achieve without compromising resiliency or throughput in a distributed system. Therefore, clients must be made aware of this behavior and their systems must be prepped to handle more than one callbacks gracefully. For instance, multiple callbacks to a client who is only sending an e-mail wouldn't be a terrible thing. A duplicate e-mail has far less consequences than, say, a monetary transaction. In the case of latter, payment gateways usually employ strict measure to execute them once and only once, so clients can rely on them to de-dupe. If client is backing up a database once a day using callback, redundant callback arriving when a back up is already in progress or have already completed once for the day can be safely ignored. Bottom line is, shortcomings of at-least once delivery model can be easily overcome within the applications on the receiving end of callbacks.

* **HTTP API latency** - It'd be prudent to set a timeout of few seconds on the http callback. Clients aren't allowed to execute any long running operations on
 the callback request synchronously, or they might end up hogging our precious and limited callback threads.

* **HTTP failures** - Clients are allowed to have transient failures as well, so such callbacks must be retried at least once before abandoned. An addition field `retryCount` in the SQS message can help keep track of retry count and add a delay b/w susequent retries using the same delay mechanism. 

## Wrapping up
In retrospect, using coroutines might look silly. But it connected the dots that led me to SQS. However, I am not under a delusion that this is the best solution to this problem. There's always more to a broad or generic design problem than what meets the eye. But until I come across a better one, this'll have to do.

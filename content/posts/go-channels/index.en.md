---
weight: 1
title: "Learning Go <- Channels"
date: 2025-12-05T00:00:00+00:00
lastmod: 2025-12-05T00:00:00+00:00
draft: false
author: "Jack"
authorLink: "https://jackroberts.uk"
description: "Understanding channels (`chan`) when learning Go"
images: []

tags: ["go", "golang", "goroutines", "chan", "channels", "concurrency"]
categories: ["post"]

lightgallery: true

toc:
  auto: false
---

## Introduction

As a Platform Engineer working exclusively with AWS day-to-day, I've found myself spending less and less time writing code. To remedy this, I decided to learn Go. One of the things I really like about Go is how it implements Concurrency - in particular, how Goroutines communicate using channels, known as the CSP (Communicating Sequential Processes) model.

Having recently stood up this site, writing a small, helpful post on this topic felt like a good way to kick things off. So below is a quick primer on Concurrency in Go - an intro to Goroutines, Channels, and how I remember which side the `<-` goes on!

## Concurrency in Go

Concurrency in Go is made up of two *things* - **Goroutines**, and **Channels**.

## Goroutines

A goroutine is effectively a lightweight thread. See below for what one looks like.

```
go func() {
      // some work you want to execute concurrently
}()
```

When you set a Goroutine off, it's running concurrently to your main function. It shares the same address space (memory) as your program, but executes independently. Because it breaks the linear flow, you can't just do `return response` within a Goroutine - there's no way to return the value that it produces, and in any case, where would it return it to? Your main program has moved on after setting the Goroutine off.

Enter **Channels**.

## Channels

A channel (`chan`) is a conduit through which you can send and receive values. You send and receive values using the `channel` operator, which is a left-pointing arrow (`<-`).

First off, the basics: 

1. There are three types of Channels - bi-directional (`chan`), send-only (`chan<-`), and receive-only (`<-chan`)
2. You will always want to initially create (`make`) a bi-directional channel by using `chan`.
3. You use the `channel` operator (`<-`) to make it send-only or receive-only in the context of how you want a function to use it.
4. The `channel` operator, i.e. the arrow, is always a left-facing arrow (`<-`), whether it's a send-only or a receive-only.

Now you know that, the only remaining question becomes - when do you use `chan`, `chan<-`, or `<-chan`? The answer, as alluded to above, comes down to restricting how you want a function to use that channel. It's like the Principle of Least Privilege, but for Go Concurrency.

> **_NOTE:_**: Even though above I describe a channel as a conduit, I found that the best way to visualize a channel is as a **pipe**.

```
      this pipe is a channel
---------------------------------

---------------------------------
```

### Bi-directional

As mentioned above, a channel can be bi-directional, simply by being passed as a `chan`. This means that the channel is two way.

You can send things through it:

```
someBiDirectionalChannel := make(chan int)

go func() {
        defer close(someBiDirectionalChannel)
        someBiDirectionalChannel <- 123 // send 123 to someBiDirectionalChannel
    }()
```

And you can receive from it:

```
for r := range someBiDirectionalChannel { // receive from the channel
      // do stuff
}
```

### Send-only

Send-only channels can be passed to a function as `chan<-`. 

Let's visualise this using the "pipe" analogy:

```
      this pipe is a channel
---------------------------------
                                  <- // data flows in to the pipe
---------------------------------
```

The arrow being at the end is the only position where it makes sense that data can be flowing **in to** the pipe. Therefore, `chan<-` is a **send-only** channel.

Below is an example where the `produce` function uses a **send-only** channel:

```
// produce accepts a send-only channel (chan<-)
func produce(pings chan<- string) {
	pings <- "hello world"
	
	// If you tried to do `msg := <-pings `
	// The compiler would throw an error: "invalid operation: <-pings (receive from send-only type chan<- string)"
}

func main() {
	messages := make(chan string)

	// Go automatically converts the bidirectional channel to send-only for the function
	go produce(messages)
}
```

### Receive-only

Receive-only channels are written as `<-chan`. 

Again, let's visualise this using the "pipe" analogy:

```
          this pipe is a channel
    ---------------------------------
 <-  // data
    ---------------------------------
```

The arrow being at the end is the only position where it makes sense that data can be flowing **out of** the pipe. It is being **received** by whatever is at the end of the pipeline, and therefore `<-chan` is a **receive-only** channel.

Below is an example where the `consume` function uses a receive-only channel:

```
// consume accepts a receive-only channel (chan<-)
func consume(pings <-chan string) {
	// We can only read from this channel
	msg := <-pings

	// If you tried to do `pings <- "some data"`
	// The compiler would error: "invalid operation: pings <- "some data" (send to receive-only type <-chan string)"
}

func main() {
	comms := make(chan string)

      // from the previous example, `produce` sends data to the channel
	go produce(comms)
	
	// Pass the same channel as receive-only
	consume(comms)
}
```

## Conclusion

And that's it - that's your very quick primer to using Goroutines and Channels in Go.

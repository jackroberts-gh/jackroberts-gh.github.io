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

As a Platform Engineer working exclusively with AWS day-to-day, I've found myself spending less and less time writing code. To remedy this, I decided to learn Go. One of the things I really like about Go is how it does Concurrency. When I initially read about Concurrency in Go (shout out to Jon Bodner's `Learning Go`), everything made perfect sense - goroutines and channels, hand-in-hand, easy.

However... I found that when I sat down to solve a problem using the language, I'd struggle to remember what sort of Channel I should be using, and which way around the arrow went. I'd find myself searching my brain for some rote memorized piece of code, rather than having a way to logically "work it out".

I'm using the above as an excuse to write this post, to help you more easily figure out whether you need a `chan`, a `<-chan`, or a `chan<-`.

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

A channel is a conduit through which you can send and receive values. 

First off, the basic rule: the `channel` operator is always a left-pointing arrow (`<-`). The only thing that changes is what side of the channel (`chan`) the arrow goes - before, or after.

Even though above I describe a channel as a conduit, I found that the best way to visualize a channel is as a **pipe**.

```
      this pipe is a channel
---------------------------------

---------------------------------
```

This way not look all that useful currently, but bear with me.

### Bi-directional

A channel can be bi-directional, simply by being declared a `chan`. This means that the channel is two way.

You can send things through it:

```
someChannel := make(chan int)

go func() {
        defer close(someChannel)
        someChannel <- 123 // send 123 to someChannel
    }()
```

> **_NOTE:_**: make(chan int) creates an unbuffered channel. This means the sender will block (pause) until someone is ready to receive the value.

And you can receive from it:

```
for r := range someChannel { // receive from the channel
      // do stuff
}
```

You might be thinking: "Great - why even bother using a `send-only` or `receive-only` channel? I'll just use a bidirectional one!" And the response would be: "Sure, go ahead, if you're happy for any Tom, Dick, and Harry to also send stuff to your channel!". 

It's idiomatic in Go to initially create a channel as a bidirectional one. However, you should only return to the caller the kind of channel you want them to work with. Do you want to give them a channel to only receive something from, or do you want to give them a channel to send stuff to? The answer is rarely both - deadlocks are a much greater risk if you do this.

### Send-only

Send-only channels are written as `chan<-`. 

Let's visualise this using the "pipe" analogy:

```
      this pipe is a channel
---------------------------------
                                  <- // data flows in to the pipe
---------------------------------
```

The arrow being at the end is the only position where it makes sense that data can be flowing **in to** the pipe.

Therefore, `chan<-` is a **send-only** channel.

### Receive-only

Receive-only channels are written as `<-chan`. 

Again, let's visualise this using the "pipe" analogy:

```
          this pipe is a channel
    ---------------------------------
 <-  // data
    ---------------------------------
```

The arrow being at the end is the only position where it makes sense that data can be flowing **out of** the pipe.

If this isn't doing it for you, you can just remember that it's the opposite/only alternative once you know that **send-only** has the arrow on the right-hand side.
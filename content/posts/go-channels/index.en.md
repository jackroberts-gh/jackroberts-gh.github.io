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

tags: ["go", "golang", "chan", "channels", "concurrency"]
categories: ["post"]

lightgallery: true

toc:
  auto: false
---

## Introduction

As a Platform Engineer working exclusively with AWS day-to-day, I've found myself spending less and less time writing code. To combat this, I've decided to learn Go.

Reading about Concurrency in Go (shout out to Jon Bodner's `Learning Go`), everything made perfect sense - goroutines and channels, hand-in-hand, easy.

However, I found that when I sat down to solve a problem using the language, I'd often find myself searching my brain for some rote memorized piece of code rather than being able to just know what I needed, which would normally come with deeper understanding.

I'm using the above as an excuse to write this post. Moreover, this post is there to help you more easily remember whether you need a `chan`, or a `<-chan`, or a `chan<-`, without having to just remember.

## Concurrency in Go

TL;DR: Concurrency in Go involves two things - **Goroutines**, and **Channels**.

When you set a **Goroutine** off and running to do some task, it's running as a separate process - it's outside of the main "synchronous" loop where your functions run in order and return values. Because of this, you can't just do `return response` - there's no way to return the value that it produces, and in any case, where would it return it to? Your main program has moved on after setting the Goroutine off.

Enter **Channels**.



## Goroutines

## Channels

First off, the basic rule: it's always a left-pointing arrow (`<-`). 

I found that the best way to visualize a channel, is as a pipe.

```
      this pipe is a channel
---------------------------------

---------------------------------
```

This way not look all that useful currently, but bear with me.

### Bi-directional

### Send-only

Send-only channels are written as `chan<-`. 

Let's visualise this using the "pipe" analogy:

```
      this pipe is a channel
---------------------------------
                                  <- // data
---------------------------------
```

The arrow being at the end is the only position where **sending** data through the channel makes sense.

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

Now.. admittedly, the analogy flouders a little here. You can interpret this as "the channel has **received** the data", and therefore, it's a **receive-only** channel. 

If this isn't doing it for you, you can just remember that it's the opposite/only alternative once you know that **send-only** has the arrow on the right-hand side.

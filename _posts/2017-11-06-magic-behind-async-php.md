---
layout: post
title: The Magic Behind Async PHP
tags: [php,amphp,async]
---
Async PHP allows a massive speedup of applications by leveraging non-blocking I/O. It allows making multiple HTTP requests in parallel or any other way of I/O multiplexing. But what's the magic behind it? How does it actually work?

## Blocking I/O

Blocking I/O is the traditional way of doing I/O in PHP. It means that any I/O request by the application is processed synchronously. The application's executing thread is suspended until the requested I/O operation is finished.

```php
// Pseudocode for connecting to a server via TLS and making an HTTP request.
$socket = connect("tls://github.com:443");
$socket->write($httpRequest);
$contents = $socket->readAll();
$httpResponse = $parser->parse($contents);
```

The application just waits for the I/O to happen. It can't do anything during that time, it just waits. Let's change that.

## Non-Blocking I/O

Non-Blocking I/O doesn't wait for the operation to finish. A non-blocking `fwrite()` either writes the contents to the send buffer or fails. But how is that useful? We need some scheduler that knows when there is enough space in the send buffer to write new contents to it, otherwise we just end up with a busy loop trying to call `fwrite()` until it succeeds and the situation is worse than with blocking I/O.

`stream_select()` to the rescue! `stream_select()` is blocking until there's some I/O activity or the timeout is reached. Instead of having the individual I/O requests block on their own, we only block once for all currently happening I/O requests. We call `stream_select()` in a loop, which is called the event loop.

```php
// A very simple event loop for I/O multiplexing.
while ($running) {
    $result = stream_select($read, $write, $except, $seconds, $microseconds);

    // invoke handlers for readable / writable streams
}
```

The event loop is the main scheduler of every asynchronous application. Usually event loops have a few more features aside from I/O events. They commonly offer features such as watching for signals, single and periodic timers.

Pending signals are usually dispatched with `pcntl_signal_dispatch()` in every loop iteration. Timers are held in a queue and the next expiring timer is picked as a timeout for the `stream_select()` call. If there's no I/O event until the timer is due to be called, the `stream_select()` call will return and the timer can be executed.

`stream_select()` is a system call that puts the current process on hold and wakes it up again once there's an event or the timeout is reached. That means we don't consume any CPU cycles while waiting for I/O events.

It's important to only have one `stream_select()` loop per application running. That means there has to be an API for registering streams to watch for I/O.

## Amp's Event Loop

[Amp](https://amphp.org/) is a framework for writing asynchronous PHP applications. It provides primitives to manage concurrency such as an event loop, promises, and asynchronous iterators. In fact, it provides multiple different event loop implementations. That's due to `stream_select()` having a fixed limit of file descriptors it can handle. There are other system calls available that can do better, but they're not exposed by PHP by default. Multiple extensions exist that allow to use these newer and better system calls, such as `ext-ev` and `ext-uv`.

Amp offers a global event loop that can be accessed by the static methods in `Amp\Loop`. It uses a global event loop, as there can anyway only be one event loop running at any time. `Amp\Loop` offers methods to register I/O and signal watchers and to register timers, just what we talked about earlier.

## Making It Easy

Manually registering I/O watchers and handling the I/O events is rather low-level. It's the way we recommended in Amp v1, but we noticed it's sub-optimal for developers. [We built a stream abstraction for Amp v2](https://github.com/amphp/byte-stream) that offers an `InputStream` and `OutputStream` interface. These allow the usage of streams without any callbacks involved.

The result of such an abstraction is that developers interact rather rarely with the event loop. On the other hand, it's still important that developers understand that the event loop is the scheduler of an asynchronous application. Knowing the basics behind "the magic" helps a lot and makes it easier to reason about asynchronous applications.

An asynchronous HTTP client can build on these basic abstractions instead of having to manually deal with the low-level readability events, just like a GitHub API client can build on an HTTP client.

## TL;DR

The magic behind asynchronous PHP is I/O multiplexing and a scheduler called the event loop. An event loop is literally just a loop that checks for events in a timer queue and using a system call to select actionable streams.

Amp is one of the available implementations in PHP that abstracts the low-level basics and has a particular focus on performance.

[Learn more by reading Amp's Getting Started guide.](https://amphp.org/getting-started/tcp-chat/)

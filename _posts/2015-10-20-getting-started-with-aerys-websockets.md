---
layout: post
title: Getting Started with Aerys WebSockets
tags: [php,amphp,async,aerys,websocket]
---
Modern web technology lets us create real time apps within the users browser. WebSockets make this communication dead simple, but traditionally, we couldn't really write performant WebSocket apps in PHP, because of the single request / response model.

Fortunately, there's a new kind of server now which has just become open source after years of work: [Aerys](https://github.com/amphp/aerys). It's an application server completely written in PHP and based on the [Amp Concurrency Framework](https://github.com/amphp/amp). If you're not familiar with Amp yet, you may want to read my last blog post ["Getting Started with Amp"](/2015/09/20/getting-started-with-amp.html) first.

> Aerys is a full-featured HTTP server, but this blog post will just cover WebSockets! There's [another blog post explaining the basics in general](http://blog.kelunik.com/2015/10/21/getting-started-with-aerys.html).

## Installation

Aerys can be installed using Composer and requires PHP 7.

```json
{
    "require": {
        "amphp/aerys": "dev-master"
    },
    "minimum-stability": "dev"
}
```

`composer install` adds the executable server file `vendor/bin/aerys`. You can now run it with the sample configuration by simply executing `vendor/bin/aerys -c vendor/amphp/aerys/demo.php`. This will boot a server listening on [`localhost:1337`](http://localhost:1337/). When visiting that address, you should see a "Hello, world."

## Configuration

Aerys is configured using a simple PHP file, no XML, no YAML! Below is the configuration file we'll use for our WebSocket chat, everything should be explained by the comments. You can find the [complete demo on GitHub](https://github.com/kelunik/demo-chat).

You can start Aerys with a specific configuration using `vendor/bin/aerys -c aerys.php`.

```
<?php

use Aerys\Host;
use Aerys\Request;
use Aerys\Response;
use Aerys\Router;
use Kelunik\Demo\Chat;
use function Aerys\root;
use function Aerys\websocket;

// Route "/ws" to the WebSocket endpoint.
// You can add more routes for "/about" or other normal pages later.
$router = (new Router())
    ->route("GET", "/test", function(Request $req, Response $resp) {
        // just another example route
        $resp->end("<h1>It works!</h1>");
    })
    ->route("GET", "/ws", websocket(new Chat));

// Add document root to serve our index.html
$root = root(__DIR__ . "/public");

// Create virtual host localhost:1337
// Requests will first be routed. If no route matches,
// the server tries to find a file in the document root.
// You can add more responders or even multiple document roots to a single host.
(new Host)
    ->name("localhost")
    ->expose("*", 1337)
    ->use($router)
    ->use($root);
```

We added a `Chat` object as WebSocket endpoint. `Chat` implements `Aerys\Websocket` and contains all logic for our chat. You can find the complete class in the [demo-chat repository on GitHub](https://github.com/kelunik/demo-chat/blob/master/src/Chat.php).

## Server Life Cycle

When the server boots up, all registered WebSocket endpoints will receive the event by a call to `onStart(Aerys\Websocket\Endpoint $endpoint)`. They can use `$endpoint` then to send messages to clients. When the server shuts down, `onStop()` will be called allowing clean up.

Normally, you won't do much more than this simple implementation when you do not have special initialization work:

```php
public function onStart(Endpoint $endpoint) {
	$this->endpoint = $endpoint;
}
```

## WebSocket Handshake

Before a WebSocket connection is established, there has to be a handshake between the client (usually a browser) and the server. `Aerys\Websocket` requires a method called `onHandshake(Aerys\Request $request, Aerys\Response $response)`. This is the place where security checks and authentication can be implemented. The most important part is to check the `origin` header, since WebSockets are not subject to the same-origin-policy. If we do not deny requests from other origins, any website could connect to our endpoint and communicate without it.

```php
public function onHandshake(Request $request, Response $response) {
    // During handshakes, you should always check the origin header,
    // otherwise any site will be able to connect to your endpoint.
    // Websockets are not restricted by the same-origin-policy!
    $origin = $request->getHeader("origin");
    if ($origin !== "http://localhost:1337") {
        $response->setStatus(403);
        $response->end("<h1>origin not allowed</h1>");
        return null;
    }
    // Returned values will be passed to onOpen.
    // That way you can pass cookie values or the whole request object.
    return $request->getConnectionInfo()["client_addr"];
}
```

If we do not alter the status code, the connection will be accepted and `onOpen(int $clientId, $handshakeData)` will be called with the return value of `onHandshake` as second parameter. When the client disconnects, times out or the server shuts down, `onClose(int $clientId, int $code, string $reason)` will be called.

## Sending / Receiving messages

Whenever a new message arrives, `onData(int $clientId, Websocket\Message $msg)` will be called. By yielding the `$msg`, you can get the message body. This is required to allow streaming messages instead of requiring the server to buffer all messages directly.

You can send messages to a single client / a group of clients or all connected clients with `$endpoint->end(int|int[]|null $clientId, string $payload)`. When `$clientId` is `null`, the message will be broadcasted to all connected clients.

A simple implementation broadcasting all messages which are received:

```php
public function onData(int $clientId, Websocket\Message $msg) {
    // yielding $msg buffers the complete payload into a single string.
    // For very large payloads, you may want to stream those
    // instead of buffering all content.

    // $msg implements Amp\Promise which updates on new content and finally
    // resolves to the full contents. Yielding an Amp\Promise in an Amp context
    // interrupts the execution and continues as soon as the promise is resolved.
    // For more information, please read the "Getting Started with Amp" post
    // mentioned earlier.
    $body = yield $msg;
	$this->endpoint->end(null, $body); // null broadcasts to all connected clients
}
```

## Conclusion

Aerys is now finally public and allows new ways of writing PHP web applications. However, it's not completely stable yet. To find inconsistencies and bad API decisions, people have to actually use it and file issues, please do!

You can find a [completely runnable version of a demo chat application on GitHub](https://github.com/kelunik/demo-chat).

## Changelog

* **Apr 16th 2016**<br>Replaced all invocations of `Response::send` with `Response::end`, since `Response::send` has been removed, as it was doing essentially the same as `Response::end`.

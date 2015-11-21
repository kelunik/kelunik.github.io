---
layout: post
title: Getting Started with Aerys
tags: [php,amphp,async,aerys]
---
[Aerys](https://github.com/amphp/aerys) is a new kind of server which has just become open source after years of work. It's an application server completely written in PHP and based on the [Amp Concurrency Framework](https://github.com/amphp/amp). If you're not familiar with Amp yet, you may want to read my previous blog post ["Getting Started with Amp"](/2015/09/20/getting-started-with-amp.html) first.

## Installation

Aerys can be installed using Composer and requires PHP 7. Since it's an application server, it makes sense to have it installed as local dependency instead of a global binary.

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

Aerys is configured using a simple PHP file, no XML, no YAML! This file contains essentially two parts, the global server configuration and a list of hosts. You can start Aerys with a specific configuration using `vendor/bin/aerys -c config.php`. Add `-d` to start it in debug mode.

### Global Server Options

Global server options are configured using a constant associative array called `AERYS_OPTIONS`. This ensures that all server configuration is at one place and not split across multiple files. Here's a list of the most important options.

Option                 | Description
-----------------------|------------
`user`                 | If Aerys is started as `root` on Linux and this value is not `null`, it switches to that user after booting.
`maxConnections`       | Maximum number of concurrent connections.
`maxKeepAliveRequests` | Maximum number of requests a client can issue using a single connection.
`maxBodySize`          | Maximum size of a request body. You may have to increase it when dealing with larger uploads.

You can find a [complete list on GitHub](https://github.com/amphp/aerys/blob/master/lib/Options.php), just use the property names as configuration array keys.

```php
const AERYS_OPTIONS = [
	"user" => "www-data",
];
```

### Hosts

Aerys supports multiple virtual hosts in a single instance. To add a new host, just instantiate a new instance of `Aerys\Host`. By default a host is exposed on port 80 and named `localhost`.

```php
(new Aerys\Host)
	->name("localhost")
	->expose("*", 80);
```

### Full Configuration Example

```php
<?php

const AERYS_OPTIONS = [
	"user" => "www-data",
];

$router = Aerys\router()->get("/", function(Aerys\Request $req, Aerys\Response $resp) {
	$resp->send("<h1>Hello World!</h1>");
});

$docroot = Aerys\root(__DIR__ . "/public");

(new Aerys\Host)
	->name("example.com")
	->use($router)
	->use($docroot);
```

## Responders

Responders are callables invoked to dispatch requests. Every host can have as many responders as needed. Responders must have the following signature:

```php
function(Aerys\Request $req, Aerys\Response $resp) { /* ... */ }
```

Responders are executed in the order they're added to the `Host` object until one responder starts outputting a response body. All following responders won't be called.

A very simple example is a responder that responds always with the same content.

```php
(new Aerys\Host)
	->use(function(Aerys\Request $req, Aerys\Response $resp) {
		$resp->send("<h1>It Works!</h1>");
	});
```

### Routing

A more complex responder is the built-in router which is based on NikiC's [FastRoute](https://github.com/nikic/FastRoute).

```php
$callable = function(Aerys\Request $req, Aerys\Response $resp, array $args) {
	$resp->send("<h1>ID: " . htmlspecialchars($args["id"]) . "</h1>");
};

$router = Aerys\router();
$router->route("GET", "/users/{id}", $callable);

$host->use($router);
```

As it's just another responder, you can always build your own router or use any other router implementation.

### Static Files

Aerys is not just an application server, but can also serve static files. The document root implementation is just another responder. Usually you want to add it as last responder, because filesystem access is slow and you want to hit your disk only if no other responder matched.

> As PHP supports only blocking filesystem access by default, you should make sure to have either pecl/eio or php-uv installed in production, see [`amphp/file`](https://github.com/amphp/file).

```php
$docroot = Aerys\root(__DIR__ . "/public");
$host->use($router);
$host->use($docroot);
```

## Advanced features

There's still more, e.g. ["Getting Started with Aerys WebSockets"](/2015/10/20/getting-started-with-aerys-websockets.html) or Middlewares which will be covered in a future blog post.

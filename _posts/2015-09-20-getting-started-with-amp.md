---
layout: post
title: Getting Started with Amp
tags: [php,amphp,async,promises]
---
[`amphp/amp`](https://github.com/amphp/amp) is a non-blocking concurrency framework for PHP. It allows writing non-blocking asynchronous code in a performant and readable way.

## Event Loop

While your script isn't calculating anything, it's usually waiting for I/O, such as a database result or external HTTP request. In an event loop with non-blocking I/O, this time can be used to allow other tasks executing their calculations. To make this possible, we need exactly two things:

* A task scheduler that executes those other tasks while one is waiting,
* and either callbacks or pause- and resumable functions.

In PHP there's no native task scheduler like we have in Javascript engines, therefore it has to be implemented in PHP. Amp provides multiple implementations of a so-called [`Reactor`](https://github.com/amphp/amp/blob/master/lib/Reactor.php). There's a `NativeReactor` that is implemented in pure PHP, but there are also other implementations using extensions underneath. There's more information about the different implementations [in the official documentation](http://amphp.org/docs/amp/reactor-concepts.html#reactor-implementations). Extension based reactor implementations are important when you're dealing with tens of thousands of watchers, but the native implementation might be faster with just a few watchers.

Regarding our second point, both options are possible with Amp. The issue with callbacks is that they're not that readable and you end up in a callback hell. With PHP's [Generators](http://php.net/manual/en/language.generators.php), which were introduced in PHP 5.5, it's not only possible to use them as iterators, but also to create pause- and resumable functions. [Nikita Popov](https://github.com/nikic) wrote a [excellent blog post about using them for cooperative multitasking](http://nikic.github.io/2012/12/22/Cooperative-multitasking-using-coroutines-in-PHP.html). Whenever you `yield` a `Promise` in an Amp context, the task scheduler will give other tasks execution time and resume when the yielded `Promise` resolves.

## Promises

`Amp\Promise` is the basic unit for concurrency with Amp. Promises are placeholders for values or tasks that aren't yet complete. You can find [more about promises](http://amphp.org/docs/amp/managing-concurrency.html#promises) in the documentation. What's important for now is just the `when` method. Basically, code using promises will look like this:

```php
$promise = asyncTask(...);
$promise->when(function($error, $result) {
	if ($error) {
		// handle exception
	} else {
		// handle result
	}
});
```

The callback will be executed when the `asyncTask` resolves and actually turns into a value or errors. Amp uses error-first callbacks for promises, because almost every async task can fail and it's too easy to miss the error case otherwise. `$result` will be the value the promise resolved to. But wait, didn't we want to avoid that callback hell? When you're in an Amp context, you can simply `yield` the promise to wait for it:

```php
$promise = asyncTask(...);

try {
	$result = (yield $promise);
} catch (MyException $e) {
	// handle exception
}
```

Isn't that a lot nicer already? It's almost like synchronous code – except for that single `yield` keyword. The advantage of using generators might become more obvious when you're dealing with multiple consecutive asynchronous tasks where you would otherwise end up with something like:

```php
$promise = asyncTask(...);
$promise->when(function($error, $result) {
	if ($error) {
		// handle exception
	} else {
		$promise = asyncTask2(...);
		$promise->when(function(...) {
			// ...
		});
	}
});
```

For more advanced use cases, there are [combinators](http://amphp.org/docs/amp/managing-concurrency.html#combinators) to wait for multiple promises at once.

## Amp context and finally a `"Hello World"`

I already mentioned using `yield` in an Amp context, but what's that actually? The task scheduler can only manage tasks it knows about. There are multiple places where generators yielding promises are automatically resolved:

### `Amp\run()`

`Amp\run` actually starts the event loop and hands over control to it. Once invoked, it keeps running until no more timer events, I/O streams or signal watchers are active or `Amp\stop` is invoked. Long-running applications generally execute entirely inside of a single `Amp\run`.

```php
Amp\run(function() {
	Amp\once(function() {
		Amp\stop(); // stops the event loop and continues with code after the Amp\run call
	}, 5000);

	Amp\repeat(function() {
		print "Hello\n"; // print a message every second until the event loop is stopped
	}, 1000);
});

// code here is not invoked until the event loop stops
```

### `Amp\resolve`

`Amp\resolve` turns an existing generator into a `Promise` and resolves it.

```php
function asyncTask() {
	$foo = yield asyncTaskFoo(...);
	// this is a workaround to allow generator returns prior to PHP 7
	yield new Amp\CoroutineResult($foo); // with PHP 7: return $foo;
}

$generator = asyncTask();
$promise = Amp\resolve($generator);
```

### `Amp\coroutine`

`Amp\coroutine` is similar, but it turns a `callable` into a `Promise` which makes it useful for callbacks which need the resolve mechanism.

```php
$subscriber->on("event", coroutine(function() {
	// amp context here
}));
```

These are the basic components for using async libraries based on Amp, but there's more if you want to write your own libaries.

## Promisors and Deferreds

`Amp\Promisor` is responsible for resolving future values once they become available. `Amp\Deferred` is the standard implementation of `Amp\Promisor`. Once an asynchronous task is invoked, a new `Deferred` object is created and the corresponding `Promise` is returned as result of the function invocation.

```php
stream_set_blocking(STDIN, false);

function prompt($message) {
	$promisor = new Amp\Deferred;

	print "{$message}: ";

    // Watch STDIN for input
    Amp\onReadable(STDIN, function($watcherId, $stream) use ($promisor) {
		$promisor->succeed(fgets($stream));
		Amp\cancel($watcherId);
	});

	return $promisor->promise();
}

Amp\run(function() {
	$username = (yield prompt("username"));
	$username = trim($username);

	print "\n\nHello {$username}!\n\n";
});
```

## Conclusion

Generators make it easier to write readable code and PHP 7 even improves them with `yield from`, `return` expressions and removes the need for parenthesis when using `yield` in an expression. Asynchronous programming with non-blocking I/O is still a very new concept in PHP, but hopefully it will gain more attention in the next few years.

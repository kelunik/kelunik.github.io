---
layout: post
title: An Introduction to Generators in PHP
tags: [php,amphp]
---
Generators have been added to PHP in version 5.5, yet they have received rather low attention. The [PHP 5.5 migration guide](https://secure.php.net/manual/en/migration55.new-features.php) introduced them as way to implement simple iterators.

> Support for generators has been added via the yield keyword. Generators provide an easy way to implement simple iterators without the overhead or complexity of implementing a class that implements the Iterator interface.

It provides a first short example of how such a simple iterator might look like, you can see it below.

```php
function xrange($start, $limit, $step = 1) {
    for ($i = $start; $i <= $limit; $i += $step) {
        yield $i;
    }
}
```

Let's compare this to a non-iterator implementation using arrays.

```php
function range($start, $limit, $step = 1) {
    $elements = [];

    for ($i = $start; $i <= $limit; $i += $step) {
        $elements[] = $i;
    }

    return $elements;
}
```

Both of these functions can be used with `foreach` to simply output all elements.

```php
foreach (range(1, 100) as $i) {
    print $i . PHP_EOL;
}
```

So what did we gain now apart from a shorter definition? What does `yield` really do? Why does the first function definition even return something, even if there's no `return`?

Let's start with the return value. Generators are special functions in PHP. Whenever a function contains `yield`, it's no longer a normal function anymore, but always returns a [`Generator`](https://secure.php.net/manual/en/class.generator.php). `Generator` implements the [`Iterator`](https://secure.php.net/manual/en/class.iterator.php) interface, that's why it works with `foreach`.

In the following code block we rewrite the previous `foreach` loop to make use of the methods of the `Iterator` interface. You can see the result on [3v4l.org](https://3v4l.org/5uF7I).

```php
$generator = xrange(1, 100);

while ($generator->valid()) {
    print $generator->current() . PHP_EOL;

    $generator->next();
}
```

We can clearly see where our generator is advanced now, let's write another generator to better understand what's happening _inside_ the generator.

```php
function foobar() {
    print "foobar - start" . PHP_EOL;

    for ($i = 1; $i <= 5; $i++) {
        print "foobar - yielding..." . PHP_EOL;
        yield $i;
        print "foobar - continued" . PHP_EOL;
    }

    print "foobar - end" . PHP_EOL;
}

$generator = foobar();

print "Generator created" . PHP_EOL;

while ($generator->valid()) {
    print "Getting current value from the generator..." . PHP_EOL;
    print $generator->current() . PHP_EOL;

    $generator->next();
}
```

```plain
Generator created
foobar - start
foobar - yielding...
Getting current value from the generator...
1
foobar - continued
foobar - yielding...
Getting current value from the generator...
2
foobar - continued
foobar - yielding...
Getting current value from the generator...
3
foobar - continued
foobar - yielding...
Getting current value from the generator...
4
foobar - continued
foobar - yielding...
Getting current value from the generator...
5
foobar - continued
foobar - end
```

Huh? Why is `Generator created` printed first? That's because generators don't execute _anything_ until you start using the returned generator. In this case it's `$generator->valid()` that starts executing the generator. We can see that the generator runs until the first `yield`. It returns control flow to the caller of `$generator->valid()` at that point. Any call to `$generator->next()` resumes the generator and any `yield` stops it again, until there are no more `yield`s. We have an interruptible function now that we can pause and resume at any `yield`. This enables writing lazy functions that do just as much as the consumer needs.

You could build something that reads all users via the GitHub API. These are paginated, but you could hide that detail and fetch new pages as needed. You could `yield` every user from the current page, as soon as there are no more users on the current page, just fetch the next page until you're at the end.

```
// This is pseudo code, but it should give you
// enough ideas to implement it with the real API.
class GitHubClient {
    function getUsers(): Iterator {
        $uri = "/users";

        do {
            $response = $this->get($uri);

            foreach ($response->items as $user) {
                yield $user;
            }

            $uri = $response->nextUri;
        } while ($uri !== null);
    }
}
```

The consumer can just iterate over all users or stop at any time.

## Generators as Iterators are Boring

Yes, you're right. What I've explained so far is the common use case people see in PHP. But iterators aren't even half of the possible use cases.

Generators also offer `send()` and `throw()` which are not part of the `Iterator` interface. We've previously talked about pausing and resuming generators. When resuming a generator, this can not only happen via `Generator::next()`, but also with `Generator::send()` and `Generator::throw()`.

`Generator::send()` let's you specify a "return value" of `yield`, while `Generator::throw()` will make the `yield` throw. With these methods we can not only pull data out of the generator, but also feed it with new data.

Let's take a look at the `logger` example from [nikic's blog post](https://nikic.github.io/2012/12/22/Cooperative-multitasking-using-coroutines-in-PHP.html
) (which is definitely worth a read).

```php
function logger($fileName) {
    $fileHandle = fopen($fileName, 'a');
    while (true) {
        fwrite($fileHandle, yield . "\n");
    }
}

$logger = logger(__DIR__ . '/log');
$logger->send('Foo');
$logger->send('Bar');
```

`yield` is used in an expression here. Whenever we send values in, they're returned from `yield` and passed as argument to `fwrite()`.

But to be honest, this example is no use case for the real world. It's useful to show how `Generator::send()` works, but just sending values in isn't really useful. It's usually better to just have a class and a normal method.

The really interesting use cases are the ones where `yield` yields a value and a "generator runner" does some work with that value and then continues the generator in the future. These include [coroutines](https://amphp.org/amp/coroutines/) and
[stateful streaming parsers](https://amphp.org/parser/).

But before we look at those use cases, let's take a quick look at `return` inside generators, which we haven't done yet. It has been possible to use `return;` inside generators since PHP 5.5, but one couldn't return values. The only purpose of `return;` was to end the generator. PHP 7.0 added the possibility to return values. This might seem strange for the iterator use case, and it is, but it's really useful for other uses cases of generators such as coroutines, i.e. when we have a generator runner that can do something with a return value instead of directly dealing with generators. We'll cover the use of `return` in coroutines in the next section.

## Generators for Async

[Amp](https://amphp.org/) is a framework for asynchronous PHP. Its coroutines yield promises, which are placeholders for future results of an operation. The "generator runner" is the `Coroutine` class. It subscribes to the yielded promises and continues the generator once the result is available. If the operation failed, it throws the error into the generator. You can [find the implementation in the `amphp/amp` repository](https://github.com/amphp/amp/blob/05491a57c98bffdd0728477819c2211c0802daeb/lib/Coroutine.php#L69-L75). A `Coroutine` in Amp is itself a `Promise`. If the coroutine throws an uncaught exception, the coroutine is failed. If it resolves successfully, it can return a value. That value is just like the return value of a normal function, but in a world with promises. That's where returning values from a generator starts to make sense and is in fact the reason why we added the possibility to PHP 7.0. Previously, we just use the last yielded value as return value, which wasn't a really good solution.

Amp allows writing non-blocking code like blocking code while opening the possibility of other non-blocking things happening concurrently in the same process. One example would be making multiple HTTP requests in parallel to one or multiple third-party APIs. But it's not limited to that. With its event loop there can be any combination of concurrent I/O operations, not just multiple operations of one kind like HTTP.

```php
Loop::run(function () {
    $uris = [
        "https://google.com/",
        "https://github.com/",
        "https://stackoverflow.com/",
    ];

    $client = new Amp\Artax\DefaultClient;
    $promises = [];

    foreach ($uris as $uri) {
        $promises[$uri] = $client->request($uri);
    }

    $responses = yield $promises;

    foreach ($responses as $uri => $response) {
        print $uri . " - " . $response->getStatus() . PHP_EOL;
    }
});
```

But coroutines with promises aren't the only use case where there's something on the right of `yield` and something "transformed" comes out at the left. Another use case are the already mentioned parsers.

```php
$parser = new Parser((function () {
    while (true) {
        $line = yield "\r\n";

        if (trim($line) === "") {
            continue;
        }

        print "New item: {$line}" . PHP_EOL;
    }
})());

for ($i = 0; $i < 100; $i++) {
    $parser->push("bar\r");
    $parser->push("\nfoo");
}
```

This parser buffers the input until there's a `"\r\n"`. Such a generator parser doesn't make things easier for such simple protocols such as a simple newline separated protocol, but they're definitely useful for more complex parsers such as a parser for HTTP requests in a server such as [Aerys](https://github.com/amphp/aerys).

## Conclusion

Generators have way more use cases than many people think. For some people a blog post like this might be the first contact with generators, some might be already using them as iterators, but there are very few usages yet using them for other things. Maybe you have even further ideas where they could be useful? I'd love comments for further ideas and hope you learned something. :-)

If you need further reading material, I recommend reading [nikic's blog post about cooperative multitasking using generators](https://nikic.github.io/2012/12/22/Cooperative-multitasking-using-coroutines-in-PHP.html).

---
layout: post
title: Parameter Type Widening in PHP 7.2
tags: [php]
---
[I have added _"Parameter Type Widening"_ to PHP 7.2](https://wiki.php.net/rfc/parameter-no-type-variance).
The [corresponding PR](https://github.com/php/php-src/pull/2265) already got a lot of negative voices saying that it would break a lot of things.
Recently there was [another blog post saying how bad it is](http://web-techno.net/php-7-2-pitfalls/).

Let me say one thing first: **Omitting the type of a parameter does not break any caller of that method.**

Quoting from the above linked blog post:

> If you expect every implementation of your interface to always have a `Tone` instance as argument, well don't expect it too much anymore.

The corresponding example looks like that:

```php
<?php declare(strict_types=1);

interface Animal {
    public function getAnimalSound(Tone $tone): AnimalSound;
}

class Tone {}


class Dog implements Animal
{
    public function getAnimalSound($tone): AnimalSound
    {
    }
}
```

The interface specifies that you can always pass a `Tone` object and will receive an `AnimalSound` instance back.
This is still true for the implementation in `Dog`.
It will accept a `Tone` object just fine.
Additionally, it will also accept anything else, maybe it can convert anything to a sound.
Note that the implementation of `Doc` clearly knows that it accepts every type, as it doesn't specify one.
The interface is just there to give the _caller_ a guarantee.

For completeness, that blog post further says:

> This is not that bad since our class `Dog` doesn't really have a contract (an interface) but only an abstract class.

The previous example in that post made `Animal` an abstract class, otherwise it just looks like the above example.
If you're not aware, there's absolutely no difference between an interface and an abstract class in terms of it being a contract that must be followed.
The identital rules apply for method signatures.

Let's have a look at a real example instead.

[`amphp/cache` has a `Cache` interface](https://github.com/amphp/cache/blob/ab2339e465d9d383dc748f288d530fd7cd7aadea/lib/Cache.php) that looks like that:

```
<?php

namespace Amp\Cache;

use Amp\Promise;

interface Cache {
    public function get(string $key): Promise;

    public function set(string $key, string $value, int $ttl = null): Promise;

    public function delete(string $key): Promise;
}
```

It uses `string` for the `$value`, which leaves serializing of complex values to the user, which is done to ensure equal behavior of all implementations.
Otherwise, some implementations might use `serialize`, some `json_encode` and some no serialization at all, because they just work in the memory of the current process.
Things might break if you switch the `Cache` implementation, hence better leave it to the user.

That package also [implements an `ArrayCache`](https://github.com/amphp/cache/blob/ab2339e465d9d383dc748f288d530fd7cd7aadea/lib/ArrayCache.php), that just stores the passed values in memory.
But that specific implementation could also support arbitrary values just fine.
It just can't because there are no contravariant argument types unless we require PHP 7.2.

This means we either have to serialize the values, which might be quite some (unnecessary) performance hit, or duplicate the implementation, removing the interface, and allowing all types there.

This works, right?
Actually, it doesn't.
The first problem here is that previously weak types would have turned an object with `__toString()` into a string, while now they just stay objects.
But we can fix that, it just needs a few `if`s.
The place where this really breaks not in the `set()` method, but rather in the `get()` method.
This isn't obvious, because types are hidden behind `Promise` placeholders, but without promises the return type would be `?string`.
So while we can accept arbitrary objects just fine, we can't ever return them, because _that_ breaks the contract.

Where is this even useful then?
Well, there are quite a few places that don't need to return the things again that they accept, or at least not in the same way.
These might include [`MessageInterface::withBody()` in PSR-7](http://www.php-fig.org/psr/psr-7/), which could automatically turn passed strings into an instance that implements `StreamInterface`.
Another important use case is adding types where they have previously only been documented, but were not language enforced.
Most userland libraries would probably just release a new major version, but that's not an option for PHP's standard library.

Quoting from the RFC:

> A change was made to the `DateTime::createFromFormat()` to add the `DateTimeZone` type to the 3rd parameter. However, this change had to be reverted, as all classes extending `DateTime` currently don't have this type declaration, so they started throwing a method signature mismatch warning. This RFC would allow the `DateTimeZone` type to be added to the 3rd parameter, without breaking code that extends the `DateTime` class.

Parameter type widening solves that.

If you want to read more on that specific topic, there are various online resources you can consult, e.g. [Wikipedia](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)#Contravariant_method_argument_type).

On the other hand, don't trust that the article too much, it for example says:

> One other instance of a mainstream language allowing covariance in method arguments is PHP in regards to class constructors. In the following example, the `__construct()` method is accepted, despite the method argument being covariant to the parent's method argument. Were this method anything other than `__construct()`, an error would occur:

PHP does not only allow covariance, it applies simply no variance rules at all to constructors.

## TL;DR

Yes, parameter type widening might break your code, but not at the place where you accept wider arguments, but at the place where you return them.

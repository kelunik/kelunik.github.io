---
layout: post
title: Running PHP tests with the highest and lowest dependencies
tags: [php,composer,tests,travis-ci,appveyor]
---

Recently, we added support for the system configuration on Windows to [`amphp/dns`](https://github.com/amphp/dns). While we had Travis running for a long while, Travis doesn't offer Windows builds. I knew AppVeyor was a thing, but I couldn't find a good example for running PHP on it, so I had a look at a [repository](https://travis-ci.org/WyriHaximus/reactphp-async-interop-loop
) of [Cees-Jan Kiewiet](https://github.com/WyriHaximus). Following that repository, [we have running tests on Windows ](https://ci.appveyor.com/project/kelunik/dns) now.

If you want to know more about running tests on Windows for PHP, [Cees-Jan Kiewiet got you covered](https://blog.wyrihaximus.net/2016/11/running-php-unit-tests-on-windows-using-appveyor-and-chocolatey/).

---

<small>**Note**: If you're trying to setup PHP on AppVeyor following that blog post just now, you might have to change `appveyor-retry cinst` to `appveyor-retry cinst --ignore-checksums` for now.</small>

---

The configuration we use for AppVeyor, tests for the lowest and highest dependencies on every version to ensure your dependency constraints are strict enough. It's something we didn't do on Travis, yet. This is how the `environment` section looks like for AppVeyor:

```yaml
environment:
  matrix:
# - dependencies: lowest
#   php_ver_target: 5.5
  - dependencies: lowest
    php_ver_target: 5.6
  - dependencies: lowest
    php_ver_target: 7.0
  - dependencies: lowest
    php_ver_target: 7.1
# - dependencies: highest
#   php_ver_target: 5.5
  - dependencies: highest
    php_ver_target: 5.6
  - dependencies: highest
    php_ver_target: 7.0
  - dependencies: highest
    php_ver_target: 7.1
```

PHP 5.5 is currently commented out because of an [open issue for Chocolatey](https://github.com/chocolatey/chocolatey-coreteampackages/issues/510). As you can see, adding a version requires adding four lines instead of one. I have not found a way to change that for AppVeyor, but it's possible on Travis.

[Cees-Jan Kiewiet covered how to get it running on Travis](https://blog.wyrihaximus.net/2015/06/test-lowest-current-and-highest-possible-on-travis/) in a similar way to AppVeyor, but there's a better way.

Travis supports a proper matrix that can be combined with the PHP versions.

```yaml
php:
  - 7.0
  - 7.1
  - nightly

env:
  matrix:
    - DEPS=lowest
    - DEPS=highest

install:
  - if [ "$DEPS" = "lowest" ]; then composer update -n --prefer-source --prefer-lowest; fi;
  - if [ "$DEPS" = "highest" ]; then composer update -n --prefer-source; fi;
  - composer show
```

This will run six jobs on Travis.

![Travis CI Builds](/img/2017/01/travis/tests.png)

Note that Travis CPU time isn't unlimited as already noted in Cees-Jan Kiewiet's blog post. If you support a lot of PHP versions ([you should not](https://blog.ircmaxell.com/2014/12/on-php-version-requirements.html)), you might want to use Travis' `include` or `exclude` options instead of additionally to exclude certain versions.

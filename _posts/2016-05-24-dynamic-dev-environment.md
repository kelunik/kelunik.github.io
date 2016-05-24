---
layout: post
title: Dynamic Developer Environments with Nginx
tags: [nginx,quicktip]
---

Did you ever want multiple developer environments without ever changing URLs? Nginx can help you. Unlike Apache, Nginx can be configuered very dynamically. You can use `map` to map cookie values to document roots.

```conf
map $cookie_environment $environment {
    default  production;
    feature  feature;
    develop  development;
}
```

You can use the `$environment` variable then in paths to `root`. You can also add a second `map` and map different environments to different ports of a `fastcgi_pass` directive.

```conf
server {
    listen 80;

    server_name example.localhost;
    index index.html;

    root /var/www/example/$environment;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

To switch environments easily in your browser, you can add bookmarks. It may not be that known, but bookmarks can be used to execute JavaScript. Simply add a new bookmark, click edit, and change the URL to something like the following:

```js
javascript:document.cookie = "environment=feature; path=/"; window.location.reload();
```

This will change your current environment and automatically reload the page. You can add a new bookmark for each environment that you have.

## Complete Example

```conf
map $cookie_environment $environment {
    default  production;
    feature  feature;
    develop  development;
}

server {
    listen 80;

    server_name example.localhost;
    index index.html;

    root /var/www/example/$environment;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

I hope this has been helpful. I would be glad if somebody could point out a similar configuration for Apache, even if it's a bit more complicated.

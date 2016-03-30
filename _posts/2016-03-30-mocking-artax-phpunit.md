---
layout: post
title: Mocking Artax with PHPUnit
tags: [php,amphp,async,artax,mocking,phpunit,quicktip]
---

[Artax](https://github.com/amphp/artax) is an asynchronous HTTP/1.1 client built on the [Amp concurrency framework](https://github.com/amphp/amp). Mocking can be great to help testing, especially for testing error cases which can be hard to receive otherwise with real requests / responses.

Mocking a HTTP response with Artax and [PHPUnit](https://phpunit.de/) is quite easy.

```php
$response = new Response;
$response->setStatus(200);
$response->setBody(json_encode([
	"type" => "urn:acme:error:badNonce"
]));

$http = $this->getMockBuilder(HttpClient::class)->getMock();
$http->method("request")->willReturn(new Success($response));
```

However, mocking complete HTTP requests can be hard, because certain headers or other things like the body might not be present in the mocked response. Sometimes you want to modify a real response just in certain details. You can do so using PHPUnit's `willReturnCallback` and `Amp\pipe`.

```php
$http = $this->getMockBuilder(HttpClient::class)->getMock();
$http->method("request")->willReturnCallback(function($request) {
    return \Amp\pipe((new Client)->request($request), function(Response $response) {
        return $response->setStatus(400)->setBody(json_encode([
            "type" => "urn:acme:error:badNonce"
        ]));
    });
});
```

Sometimes, you want to have multiple requests to the same client, but just mock certain requests. In that case, you can modify responses based on the original request and return all other responses as is.

```php
$http = $this->getMockBuilder(HttpClient::class)->getMock();
$http->method("request")->willReturnCallback(function($request) {
    return \Amp\pipe((new Client)->request($request), function(Response $response) {
        $request = $response->getRequest();

        if ($request instanceof Request && $request->getMethod() === "POST") {
            return $response->setStatus(400)->setBody(json_encode([
                "type" => "urn:acme:error:badNonce"
            ]));
        }

        return $response;
    });
});
```

You can see an [example in my ACME library](https://github.com/kelunik/acme/blob/dde5598e18f4a712d3ff9498fcb3419fc9fa8a23/test/AcmeClientTest.php#L177-L233).

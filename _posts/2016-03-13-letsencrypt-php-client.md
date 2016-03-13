---
layout: post
title: Let's Encrypt with PHP
tags: [php,ssl,encryption,letsencrypt]
---

[Let's Encrypt](https://letsencrypt.org/), the free and automated CA, started late last year into their public beta. They offer a pretty comprehensive client to automate the process of getting SSL clients and installing them. It will soon be [moved to a new home at EFF](https://letsencrypt.org/2016/03/09/le-client-new-home.html).

If you're already using PHP and don't need / want the automatic installation of your free certificates or don't want to install Python, [here's a client for you](https://github.com/kelunik/acme-client).

[`kelunik/acme-client`](https://github.com/kelunik/acme-client) is able to issue and renew certificates using your already running webserver (e.g. Nginx or Apache). It's using the `http-01` challenge to prove your domain ownership / control to Let's Encrypt. If you want to know more about `http-01` and other challenge types, [read the current draft of the ACME specification](https://ietf-wg-acme.github.io/acme/).

## Registering with Let's Encrypt

Before you can issue your first certificate, you have to create an account. All domain authorizations and certificates you issue will be bound to that account.

To register a new account, use the following command:

```plain
bin/acme setup -s letsencrypt --email me@example.com
```

Be sure to add an e-mail to get expiration reminders and other important notifications related to SSL.

## Issuing a New Certificate

As soon as you registered with Let's Encrypt, you're ready to issue your first certificate. Be sure your webserver is up and running.

Let's Encrypt currently limits certificate issuance to five certificates per week. You might want to use their staging server for testing first, which has much higher rate limits. Use `letsencrypt:staging` instead of `letsencrypt` as server (`-s`).

```plain
bin/acme issue -s letsencrypt -d example.com:www.example.com -p /var/www
```

You can separate multiple domains (`-d`) as well as multiple paths (`-p`) by separating them with colons. If you specify less paths than domains, the last one will be used for the remaining domains. The client will request a challenge for each domain and will try to solve it. Once all challenges are solved, the client requests the certificate to be issued for all these domains and saves it to  `./data/certs/acme-v01.api.letsencrypt.org.directory/example.com`.

The first domain will be used as common name and as name for saving it to the file system. There will be multiple files in the directory:

 * `cert.pem`<br>The certificate itself. Suited for servers like Apache.
 * `chain.pem`<br>The certificate chain. Usually contains one intermediate certificate, might contain more for other CAs. Suited for servers like Apache.
 * `fullchain.pem`<br>The certificate ifself concatenated with the certificate chain. Suited for servers like Nginx.
 * `key.pem`<br>The private key.

## Automating the Process

Certificates issued by Let's Encrypt are valid for 90 days. That's a rather short time, it urges to automate the process.

Theoretically, you could just setup a [cron](https://en.wikipedia.org/wiki/Cron) process, that runs every 60 days. 60 days, so you have enough time to fix errors in case the automation fails. But that's a rather bad approach, because your automation would fail in case Let's Encrypt had a maintenance during the few minutes where the client is run. The next execution would be 60 days later, so 30 days too late.

The client has a simple check mechanism to make renewal easier. You can use a cron that runs daily to check your certificates, only if they're going to expire soon, it will run the `issue` subcommand to renew the certificate.

```plain
bin/acme check -s letsencrypt --name example.com || bin/acme issue ...
```

`--name` is the common name of the certificate. You can use `--ttl` to specify a time interval in days until the certificate will expire. So `--ttl 10` will renew, when the certificate is no longer valid than 10 days. Default value is `30`.

Be sure to reload your webserver configuration afterwards, so it reloads the new certificates.

This mechanism prevents renewing too often, but ensures that automation doesn't fail that easily. If automation fails nevertheless, there are still the expiration notices you'll receive per e-mail.

## Revoking a Certificate

Last but not least, there are times where you want to revoke a certificate. This is mostly due to (potentially) compromised private keys.

```plain
bin/acme revoke -s letsencrypt --name example.com
```

As for `check`, `--name` is the common name of the certificate to revoke.

Note that revoking a certificate doesn't reset the rate limit as you might expect. Let's Encrypt still has to sign OCSP responses for revoked certificates, so revocation doesn't decrease their load.

## Support

If you need help with the client, just leave a comment below. There's also a built-in help in the client, use `bin/acme --help` or `bin/acme subcommand --help`.

---
layout: post
title: Comment on Namecheap's SSL
tags: [ssl,encryption,comment]
---

Namecheap published a [blog post recently explaining the difference between Namecheap's paid SSL and other free CAs](https://blog.namecheap.com/ssl-from-namecheap-whats-the-difference/). They said they want to cover the facts, so I'll write about the actual facts here.

I added quotes everywhere and took their headings to make it easy to read the corresponding parts in the original blog post.

## Free SSL is Encryption Only

> Domain Validation certificates are excellent products, but free DV certs differ from purchased DV certs in key areas.

**No**, free DV certificates do not differ in any way from paid DV certificates, apart from the price of course. The only thing that's checked there is the ownership of the domain, nothing more.

> Though all SSL provides encryption of data as it passes between websites, free certs provide encryption only.

**No**, DV certificates provide authentication that you're talking with the server you intended to talk to, identified by the domain name. That's why it's called domain validation.

> Free Certifying Authorities (or “CAs”) perform no validation checks on the applying business prior to issuance.

**Oh**, so we're now talking about OV (organization validation) and EV (extended validation)? That's an entirely new point.

> Unfortunately, as the issuing CA has no control over the site on the other side of the certificate, ...

**Yes**, CAs have no control over sites they issue certificates for, that's good.

> ... it’s possible that phishing sites and malware scammers will use free SSL to appear legitimate, due to the quick and automated way free certificates are issued.

**Yes**, that's possible, but also free CAs check for blacklists currently. In general, CAs issuing DV certificates should do their job and validate if the requester is in control of a domain, not fight in the front line against phishing. There's a good [blog post by Josh Aas in the Let's Encrypt blog](https://letsencrypt.org/2015/10/29/phishing-and-malware.html) if you want to read further.

If you want free SSL with encryption only, use self-signed certificates and don't add them to your trust store. *Then* you have encryption-only SSL.

## The importance of Validation

> This is a key distinction – the true value of purchased SSL security arises from knowing that the owner of the cert is who they say they are, not simply that they have control of the domain they applied with.

**Yes**, absolutely right. DV checks the domain ownership, OV and EV check the actual organization behind.

The following points are all specific to LE (short for [Let's Encrypt](http://letsencrypt.org)), a currently in closed beta CA that will issue free DV certificates.

> Short lifetimes – Free certs have 90-day lifetimes, standard. Though future automation may eliminate the need to renew every 90 days, short lifetimes will always mean more potential points of failure.

This is totally wrong and currently only specific to LE. Other CAs offering free certificates do issue one year certificates like [StartSSL](http://www.startssl.com/) or [WoSign](http://www.wosign.com/english/freessl.htm).

> No wildcards – Free certificates secure single domains only. They cannot secure multiple subdomains of a domain.

**Right**, there is no wildcard support in the current [ACME spec](https://github.com/ietf-wg-acme/acme), but this doesn't mean that it's not possible in the future. Once added to ACME, CAs like LE will probably support it.

**Wrong**, they can secure multiple subdomains, currently LE's limit is 100 domain names per certificate.

> Limited flexibility – To use free SSL, you must have root access to your servers and install and run code as a super-user.

**Wrong**, there are other ACME clients like [no-sudo](https://github.com/diafygi/letsencrypt-nosudo) and there's always the option of a manual issuance with the official LE client.

> Additionally, free CAs place restrictions on issuing certs for domains that have existing certs from other CAs.

**You totally didn't get the point of that**. This is LE specific and is a protection that all CAs should probably have, would really reduce the number of mis-issued domains. You can always issue certificates, even if there's already a service running by proving prove of the ownership of the corresponding private key.

## Build Trust with Purchased SSL

The paragraph is basically fine. Use OV and EV certificates and pay for them to improve trust and reduce the risk of forgery. But:

> The levels of encryption, validation, and trust that business and commerce websites require are available only with purchased SSL.

**Wrong**, the encryption doesn't depend on the certificate at all.

## Conclusion

Seems like this post tries to defend Namecheap's SSL business while arguing against Let's Encrypt without even mentioning it. [Let's Encrypt will be in public beta on December 3rd](https://letsencrypt.org/2015/11/12/public-beta-timing.html). I hope it will increase the use of encryption and authentication in our today's internet and make it more secure.

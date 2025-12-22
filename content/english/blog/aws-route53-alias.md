---
draft: false
author: "David Lakatos"
title: "Advanced DNS capabilities of AWS Route 53 - Part 1"
description: "Showcasing the power of Route 53's ALIAS RRtype."
publishDate: 2025-12-22
date: 2025-12-22
Image_webp: images/blog/aws-route53-alias-cover.webp
image: images/blog/aws-route53-alias-cover.png
tags:
  - aws
  - dns
  - migration
  - cloud
---

The Domain Name System is a true relic among all of the communication protocols out there. Its first appearance was back in 1983 related to RFC 882, but many-many extensions were built upon this dinosaur. This article series focuses on special features that differentiate DNS services today.

Let's imagine that you have built a single-page application (SPA) using your favourite JavaScript framework. The front-end is used by millions all over the World. It relies on a RESTful public API hosted by a 3rd party provider. Your application's build pipeline outputs static files like HTML, images, and generated JavaScript source code. Hosting the application requires only a geo-distributed Web server, so you decide to deploy it to the AWS CloudFront content delivery network (CDN).

You prefer to enable users to access the application via both `example.com` and `www.example.com` domain names. Since this is a Web application, its conventional DNS name should be `www.example.com`. The following `example.com` zone records are created to route clients to the distribution:

```
Key              RRType  Value                                                    TTL
--------------------------------------------------------------------------------------
example.com.     SOA     ns.myreg.com. hostm.myreg.com. 1 7200 900 1209600 86400  900
example.com.     CNAME   abcdefgh.cloudfront.net.                                 1800
www.example.com. CNAME   abcdefgh.cloudfront.net.                                 1800
...
```

The `SOA` record describes the `example.com.` zone's authority settings, the first `CNAME` routes `example.com` traffic to `abcdefgh.cloudfront.net`, while the second `CNAME` routes the rest of the traffic to the same CloudFront distribution.

You realize, that your DNS does not accept this record:
```
example.com.     CNAME   abcdefgh.cloudfront.net.                                 1800
```

The reason is that creating a `CNAME` record for the domain root `example.com.` is not compliant to any RFC specifications, your DNS provider was right to deny your zone configuration. What can you do in this situation?

1. Accept the fact, that `example.com` will never route clients to your app.
1. Install a Web server that HTTP redirects clients from `example.com` to `www.example.com` at the cost of extra infrastructure components, increased client load times due to the redirect, and lower system availability caused by extra point of failure.
1. Migrate your DNS service to a provider that has a trick up its sleeve.

Option 3 is a non-standard DNS service functionality called the `ALIAS` record type (aka. apex `CNAME` or flattened `CNAME`). Various DNS service providers may have different implementations, or no implementation at all. An `ALIAS` record flattens the `CNAME` hierarchy abstracting away the real content of the queried zone. A client query is served in the following manner:

![ALIAS RRtype resolution](/images/blog/aws-route53-alias.svg)

The `example.com` zone's authoritative DNS server queries the `cloudfront.net` zone for the `ALIAS`ed records. When queried, the `example.com` zone's authoritative DNS server returns `abcdefgh.cloudfront.net` zone's respective `A` and `AAAA` RRtype record values as if they were registered in the `example.com` zone.

Furthermore, AWS takes `ALIAS`ing to a whole new level. Route 53 DNS service and many AWS services such as CloudFront CDN is tightly integrated with each other. A Route 53 `ALIAS` record can specifically refer to a CloudFront distribution by their identifier. This means, that `ALIAS` records are not just dangling zone records containing any string value, but they interconnect services semantically in your AWS ecosystem. This integration is a feature out of many which transforms Route 53 from a plain DNS service into an elegant platform solution.

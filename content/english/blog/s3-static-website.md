---
draft: true
author: David Lakatos
title: "How it's made: static website running on AWS"
description: Case study how sysagnostic runs its own Hugo based static website on the Amazon Web Services cloud landscaped by OpenTofu.
publishDate: 2025-01-04
date: null
Image_webp: images/backgrounds/hero-area.webp
image: images/backgrounds/hero-area.jpg
tags:
  - aws
  - s3
  - hugo
  - opentofu
---

We have to live with our decisions; no matter if they are good or bad. This post is a showcase of our company focus: cost-effectiveness. Let me show you how we designed our website infrastructure, explain the decisions we made, how they will hopefully benefit us in the long run.

## Why go static?

Our company created its website with a set of well-specified business intentions in mind:

- communicate our professional services,
- introduce our team,
- provide contact information.

Really, nothing fancy there. We expect no new functionality requirement in the future that would complicate its design principles.

We follow the [KISS](https://en.wikipedia.org/wiki/KISS_principle) principle as all security focused engineers should do. Less unnecessary functionality translates to higher quantity and lower severity threat vectors. If our goals may be achieved using a static website, there is no security-aware reason to go with a dynamic, probably more vulnerable one.

In theory, there are three kinds of costs when running a website in the cloud:

1. run stuff,
2. store stuff,
3. transmit stuff.

A dynamic website serves content by running queries in databases, generates content dynamically per request, implements business logic on the server side, etc. Dynamic websites generate all three kinds of cloud costs keeping the TCO high:

1. run stuff,
2. store stuff,
3. transmit stuff.

Running stuff is definitely the most expensive part of the three cost categories. If we can get rid of it, the TCO may be reduced significantly. Since static websites implement all business logic on the client side via JavaScript, and - as its name suggests - only static content is served, there is no need to run stuff for a static website. When implementing a static website, only these two cost categories will generate its TCO:

1. store stuff,
2. transmit stuff.

No EC2 instances, no ECS containers, no running component at all that we should pay for by the hour.

## Website development

We are a DevSecOps company. Website development is something we are able to do for internal purposes, but it's definitely not our strong suit. Let's keep it short then, here's a rundown of how our website was born:

- [Hugo - The world’s fastest framework for building websites](https://gohugo.io/)
- [Meghna Hugo theme](https://github.com/themefisher/meghna-hugo)
- CSS magic, based on company graphic design principles

Check out the website's repo on GitHub: https://github.com/sysagnostic/website.

## Cloud infrastructure

We love AWS as you may have seen in our [professional service portfolio](https://www3.sysagnostic.com/#portfolio). It's a no-brainer that our website should "run" in AWS. Or should we say it should "sit" in AWS, since we shall have no running components.

We shall only store and transmit stuff. S3 object storage is one of the most ancient and reliable services of AWS released in 2006. It became the obvious choice for our use-case. We store the website's static web content in an S3 bucket.

### Access
AWS S3 buckets are by default region locked. This means that object access latency and data transfer speed would severely depend on the distance and network quality between the S3 bucket's region and the website visitor's geo-location.

By using AWS CloudFront CDN caching, we bring our website geographically closer to the visitor - wherever they may reside on the globe. Object access latency and data transfer speed are more deterministic, achieving a significantly better visitor experience around the world.

### Costs
Accessing S3 content - whatever access tier the object resides in - costs you a non-negligable amount of money per 1,000 request and per GB according to [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/). We should avoid a solution that has direct connection between end-user HTTP requests and S3 access requests.

AWS CloudFront's caching functionality is the cost-effective solution for our static website. Although it also has a per-request cost according to [Amazon CloudFront pricing](https://aws.amazon.com/cloudfront/pricing/) after surpassing a generous free-tier offering, the egress data transfer costs (to the Internet) are only a fraction compared to S3.

In short:

- S3 does not have an egress free-tier data transfer plan,
- CloudFront has an egress free-tier data transfer plan,
- S3 egress data transfer costs around <u>1,000 times more</u> per GB, even when compared to CloudFront's non-free-tier prices.

![CDN-distributed static website](/images/blog/how-its-made-cloudarch.svg)

Since our company provides global services, we do not apply CloudFront price class restrictions to the website's distribution. Our website may be accessed with a similar latency from the USA, Germany, Hungary or Japan. This means by applying our non-discriminating CDN policy, we accept the fact that our website's TCO may become higher if visited regularly from eg. Japan, which belongs to a more expensive country as per CloudFront's pricing policy.

### Our region of choice
So CloudFront CDN serves our website, and we store the geo-cache origin files in S3. As a European company, we opted for storing our data within the borders of the European Union. This means we could place our S3 bucket in the following AWS regions:

- Stockholm
- Frankfurt
- Ireland
- Milan
- Paris
- Spain

As per a quick comparison, the following regions provide the most cost-effective S3 service:

- Stockholm
- Ireland

Since we are using CloudFront CDN, it does not matter where we store our data, regarding data transfer speed and latency to the end-user. On the other hand, our engineers will access the bucket directly from time to time, eg. to upload new content manually for testing purposes. They are all geographically located closer to Stockholm, so we eventually chose the Stockholm AWS region to place our S3 bucket into.

### Security
Let's see how we protect our website when we store and transmit it in AWS. S3 offers an option to serve web content directly from the bucket over HTTP. Due to cost and performance reasons, this feature shall not be used. 

Only allowed services may access the bucket directly, so the `Block all public access` security failsafe shall be enabled for the bucket. Since AWS CloudFront caches the site content directly from S3, it must be able to access objects via S3 API's `GetObject` operations. 

Let's create a resource policy for the bucket that allows specifically one CloudFront distribution to access any object in the bucket:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::www.sysagnostic.com/*",
            "Condition": {
                "ForAnyValue:StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/ABCDEFGHIJKLMN"
                }
            }
        }
    ]
}
```

After applying the resource policy, we should invalidate the whole CloudFront cache. Negative object retrievals may have been cached already due to the lack of `GetObject` permission. These failed retrievals will be tried again at first access:
```
$ aws cloudfront create-invalidation --distribution-id ABCDEFGHIJKLMN --paths '/*'
```

## CI/CD
We believe you should automate everything. The website's cloud infrastructure, including the prod and staging environments, is no exception to this philosophy. The infrastructure automation has been implemented by IaC with OpenTofu technology. We store the infrastructure state in an S3 bucket for distributed access.

Check out the website's IaC repo on GitHub: https://github.com/sysagnostic/website-tofu.


We use GitHub Actions to automate all repetitive tasks related to the website software development lifecycle:

1. generate static content with Hugo,
1. deploy generated content to selected S3 bucket,
1. invalidate CDN cache.

Whenever the applied Gitflow Workflow produces a new release tagged on the `master` branch, a new deployment is scheduled to the prod S3 bucket, and the prod distribution is invalidated as the last step. Other branches deploy staging builds for UAT purposes to different buckets on a per-commit basis.

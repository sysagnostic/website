---
draft: false
author: "David Lakatos"
title: "AWS re:Invent 2024 re:Cap"
description: "Takeaways of an AWS re:Invent 2024 re:Cap meetup"
publishDate: 2025-02-23
date: 2025-02-22
Image_webp: images/blog/aws-reinvent-2024-recap.webp
image: images/blog/aws-reinvent-2024-recap.png
tags:
  - aws
  - meetup
  - reinvent
  - recap
---

Amazon once again introduced hundreds of new features for their public cloud platform, Amazon Web Services. Sysagnostic attended a re:Cap [meetup](https://www.meetup.com/aws-serverless-budapest/events/305715582/), which is an event based on the original re:Invent 2024. The re:Cap was organized by [SnapSoft](https://snapsoft.io/), and took place in the SnapSoft headquarters. As our company focuses on cost-efficient solutions, this article hand-picks two topics related to cost efficiency that were mentioned during the 2024 re:Cap meetup or during the AWS re:Invent event.

## Aurora Serverless v2
The workload for an AWS Aurora Serverless database may plummet to zero for hours or even days. A good example would be a UAT database instance that is only used during business hours. There is no need to use nor pay for a service outside business hours, but compared to other serverless services, you have had to pay for the database service's non-zero Aurora Capacity Units (ACU) anyways, until recently.

Since late 2024, Amazon Aurora Serverless v2 supports [automatic pause and resume capability with 0 ACU](https://aws.amazon.com/blogs/database/introducing-scaling-to-0-capacity-with-amazon-aurora-serverless-v2/) resource consumption. The database service is now suspended whenever the active connection count drops to zero for the configured time period. Using this feature comes at a cost of ~15s cold-start delay during establishing the first user connection to the suspended database. This approach to cut operational costs for development and test databases fits most business use-cases better than scheduled start and stop triggers, since the suspend action is tied to the fact of idling, and not statically configured, workhour-based triggers that may vary without notice.

Adrián Mezei, AWS Cloud Architect at SnapSoft, performed an impressive hands-on demo during the event, showcasing how 0 ACU serverless services cold-start with the expected resumption delay. Any Aurora Serverless v2 database service that runs a somewhat recent version of PostgreSQL or MySQL compatible instance may be reconfigured to suspend after a certain idle time period.

## Amazon FSx for OpenZFS
The re:Cap event unfortunately did not, but re:Invent 2024 did cover the newly announced [Amazon FSx for OpenZFS Intelligent-Tiering](https://aws.amazon.com/blogs/aws/announcing-amazon-fsx-intelligent-tiering-a-new-storage-class-for-fsx-for-openzfs/) storage class. Intelligent-tiering may sound familiar for anyone who worked with AWS S3 before. The goal is pretty much the same as for S3: automatic cost-saving based on data usage patterns. Data usage patterns are evaluated a bit differently here than for S3. For FSx OpenZFS, they are merely metrics on last usage. When using intelligent-tiering, AWS moves your data between frequent access, infrequent access, and archive tiers automatically, depending on the last time you accessed your data. Frequently accessed data is always kept on SSD, while less frequently accessed data is moved to different kinds of HDD. Durability obviously never suffers during and after tier changes, and interestingly, AWS guarantees the same time-to-first-byte latency, no matter which tier your data currently resides. Compared to the other storage class called provisioned, where your data always resides on SSD, intelligent-tiering is an elastic class which should be used from now on for common use-cases.

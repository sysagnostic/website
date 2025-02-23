---
draft: true
author: "David Lakatos"
title: "AWS re:Invent 2024 re:Cap"
description: "Takeaways of an AWS re:Invent 2024 re:Cap meetup"
publishDate: null
date: 2025-02-22
Image_webp: images/blog/aws-reinvent-2024-recap.webp
image: images/blog/aws-reinvent-2024-recap.png
tags:
  - aws
  - meetup
  - reinvent
  - recap
---

Amazon once again introduced hundreds of new features for their public cloud platform Amazon Web Services. Sysagnostic attended a re:Cap [meetup](https://www.meetup.com/aws-serverless-budapest/events/305715582/) based on the original re:Invent 2024 event. The re:Cap was organized by [SnapSoft](https://snapsoft.io/) and took place in the SnapSoft headquarters. Our company focuses on cost-effective solutions, so this article hand-picks the cost-effectiveness related topics that were mentioned during the 2024 re:Cap meetup or during the AWS re:Invent event.

## Aurora Serverless v2
An AWS Aurora Serverless database transaction loads may plummet to zero for hours or even days. A good example for this case would be a UAT database service that is only used during business hours. There is no need to use nor pay for the service outside business hours, but compared to other serverless services, you did pay for the database service's non-zero Aurora Capacity Units (ACU) anyways, since there.

Since late 2024, Amazon Aurora Serverless v2 supports [automatic pause and resume capability with 0 ACU](https://aws.amazon.com/blogs/database/introducing-scaling-to-0-capacity-with-amazon-aurora-serverless-v2/) resource consumption. The database service is suspended, whenever the active connection count drops to zero for the configured time period. Using the feature comes at a cost of ~15s cold-start delay during establishing the first user connection to the suspended database. The described approach to save on development and test database operational costs fits better most business use-cases compared to scheduled start and stop triggers since the suspend action is tied to the fact of idling not statically configured work hour based triggers that may vary without notice.

Adrián Mezei, AWS Cloud Architect at SnapSoft, performed an impressive hands-on demo during the event, showcasing how 0 ACU serverless services cold-start with the expected resumption delay. Any Aurora Serverless v2 database service that runs a somewhat recent version of PostgreSQL or MySQL compatible instance, may be reconfigured to suspend after a certain idle time period.

## Amazon FSx for OpenZFS
The re:Cap event unfortunately did not, but re:Invent 2024 did cover the newly announced [Amazon FSx for OpenZFS Intelligent-Tiering](https://aws.amazon.com/blogs/aws/announcing-amazon-fsx-intelligent-tiering-a-new-storage-class-for-fsx-for-openzfs/) storage class. Intelligent-tiering may sound familiar for anyone who worked with AWS S3 before. The goal is pretty much the same as for S3: data usage pattern based automatic cost-saving. Data usage patterns are evaluated a bit differently than for S3, they are merely last usage based metrics for FSx OpenZFS. When using intelligent-tiering, AWS moves your data between frequent access, infrequent access, and archive tiers automatically depending on the last time you accessed your data. Frequently accessed data is always kept on SSD, while less frequently accessed data is moved to different kinds of HDD. Durability obviously never suffers during and after tier changes, and interestingly, AWS guarantees the same time-to-first-byte latency no matter in which tier your data currently resides. When compared to the other storage class called provisioned, where your data always resides on SSD, intelligent-tiering should be used from now on for common use-cases.

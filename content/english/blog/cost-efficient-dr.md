---
draft: false
author: David Lakatos
title: Cost-efficient disaster recovery strategy
description: The cheapest infra is the one that you don't have. A sneak peek at a disaster recovery strategy that is cost-efficient without any significant downsides.
publishDate: 2025-04-30
date: 2025-04-20
Image_webp: images/blog/cost-efficient-dr-cover.webp
image: images/blog/cost-efficient-dr-cover.jpg
tags:
  - aws
  - disaster-recovery
  - cost-efficiency
  - pilot-light 
  
---

We tend to design business-critical systems to deal with extraordinary situations: natural disasters, human-induced incidents/accidents near data centers, or even acts of war targeting compute infrastructure. These occasions happen randomly from the designer's perspective, so we need to have redundancies in place all the time to be able to deal with the fallout.

Disasters are such severe events by definition that may render entire system operations – including human and machine resources inoperable in a data center. Disaster recovery plans must assume that any geographically co-located resources may be destroyed together, or any network physically co-located trunk of cables may be severed together, in an instant. Primary and disaster recovery infrastructure resources must not have common points of failure by design.

We are going to look at five different DR strategies with an increasing level of cost-efficiency, and the trade-offs between costs and other non-functional properties such as recovery point objective (RPO), recovery time objective (RTO), and performance degradation in case of a disaster.

## Backup & restore 
![Backup & Restore DR strategy](/images/blog/cost-efficient-dr-strategies-backup-restore.svg)

The backup & restore strategy requires you to back up your entire infrastructure frequently. Recovery is troublesome, a lot of data – created after the last backup – may be lost during the process.

## Warm standby
![Warm Standby DR strategy](/images/blog/cost-efficient-dr-strategies-warm-standby.svg)

The warm standby strategy provides great recovery objectives. Unfortunately, it continuously consumes a lot of resources, making it less desirable due to its low cost-efficiency.

## Cold standby
![Cold Standby DR strategy](/images/blog/cost-efficient-dr-strategies-cold-standby.svg)

The cold standby strategy is an improved version of warm standby strategy regarding cost-efficiency, while its recovery objectives are almost as good as warm standby strategy's. Due to a still significant, continuous resource consumption, its 2-site upkeep costs, and disaster performance degradation characteristics, this strategy remains a suboptimal choice for enterprise disaster recovery planning.

## Active–active
![Active–Active DR strategy](/images/blog/cost-efficient-dr-strategies-active-active.svg)

When using the active–active strategy, all the consumed resources are actively used by the system. There are still two remaining downsides to this strategy, which are the 2-site upkeep costs, and its disaster performance degradation characteristics.

## Pilot light
![Pilot Light DR strategy](/images/blog/cost-efficient-dr-strategies-pilot-light.svg)

The pilot light disaster recovery strategy keeps your disaster recovery infrastructure upkeep costs at bay. [OpenTofu](https://opentofu.org/) (formerly Terraform) is able to build up and start your entire IT infrastructure in the cloud within minutes. The strategy keeps your data (configuration, database, file system) in sync with your cloud storage. You pay for negligible storage and network resources during business-as-usual hours, and possibly pay for no compute resource at all. Disasters increase your cloud costs temporarily, so pilot light is truly beneficial for companies where disasters are rare events – as they should be.

Whenever the pilot light strategy is applied, all consumed resources are used by the system. As opposed to the other strategies, only a single site's upkeep is added to the infrastructure costs. If desired, the pilot light strategy is able to handle a disaster without performance degradation.

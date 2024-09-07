---
author: ["Ziheng Zhang"]
title: "Master Thesis: High Availability in Lifecycle Management of Cloud-Native Network Functions"
date: "2024-09-05"
description: "A Near-Zero Downtime Database Version Change Prototype"
summary: "A Near-Zero Downtime Database Version Change Prototype"
tags: ["Kubernetes", "Docker", "Golang"]
categories: ["Tech"]
series: ["Master Thesis"]
ShowToc: true
TocOpen: false
cover:
  image: "https://github.com/user-attachments/assets/ee6aca7f-a52f-4602-a3e1-4d2fbcfb24bb"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
---

This blog introduces my master's thesis ([[pdf]](https://www.diva-portal.org/smash/record.jsf?aq2=%5B%5B%5D%5D&c=1&af=%5B%5D&searchType=UNDERGRADUATE&sortOrder2=title_sort_asc&language=en&pid=diva2%3A1781462&aq=%5B%5B%7B%22author%22%3A%5B%22Zhang%2C+Ziheng%22%5D%7D%5D%5D&sf=all&aqe=%5B%5D&sortOrder=author_sort_asc&onlyFullText=false&noOfRows=50&dswid=733) & [[presentation]](https://drive.google.com/file/d/1JTO0_wDDCrYbJaCS6PHPoJIcsNHOAGO2/view?usp=sharing)) and the demo video can be found below. Most of the sections correspond to paragraphs from the thesis, while some parts that are necessary in the thesis but not needed in the blog have been omitted. This allows you to focus more on the project itself.

{{< youtube L9WOx1jLnF0 >}}

## Background

In any infrastructure, be it clustered or otherwise, databases often face challenges with version changes, including upgrades and downgrades. Upgrades are straightforward, driven by the need to adopt new features or improvements. However, downgrades, especially in the telecommunications industry, present unique challenges. For example, in the Access and Mobility Management Function (AMF) as shown in the image below, when a database is upgraded and data is converted to a new version, complications may arise. According to ETSI MANO specifications, a rollback to the previous version may be required in such cases.

{{< custom-img src="https://github.com/user-attachments/assets/9d7f0aa5-16b3-4da7-a8f6-1c4880c3ec6a" cap="3GPP 5G Core Architecture" >}}

Another scenario is the "observation period" following an upgrade. For example, after upgrading a service from version A to version B, its performance is monitored. Key questions arise: Is the service functioning correctly? Is it handling the expected load? If the service underperforms, a rollback may be required, presenting the same challenges as in the first scenario. Both cases highlight the need for better support for database downgrades to mitigate service disruptions and maintain user experience.

## Kubernetes Operator

A Kubernetes Operator (K8s Operator) encodes human operational expertise into software. The image below illustrates the key differences between using a K8s Operator and not.

{{< custom-img src="https://github.com/user-attachments/assets/4ce9297a-fbbd-4823-b67b-4af5bfc0ff7c" cap="Comparison of Cluster Management: Manual vs. K8s Operator" >}}

Without a K8s Operator, maintainers must deeply understand how stateful applications work within the cluster, execute commands in the correct order, and manually verify responses.

With a K8s Operator, maintainers can perform complex operations, like database upgrades or downgrades, by simply modifying the custom resource definition (YAML file), without needing to understand the internal mechanisms.

In these scenarios, the K8s Operator acts like an experienced maintainer, monitoring the cluster 24/7 and exposing a simple API, greatly reducing the chance of human errors.

### Zalando Postgres Operator

[The Zalando Postgres Operator](https://github.com/zalando/postgres-operator/tree/master) is a widely-used tool in the industry, known for supporting both minor and major version upgrades.

Minor upgrades, such as moving from version 1.0 to 1.1, are straightforward and can be performed without downtime. However, major upgrades, like transitioning from version 1.0 to 2.0, present more challenges.

The Zalando Operator offers two upgrade methods:

* Upgrade via Cloning: This approach requires the new cluster to run a higher version than the source. It may involve significant downtime, as all write operations must stop, and Write-Ahead Log (WAL) files must be archived before cloning begins.
* In-place Major Version Upgrade: While more convenient, this method is irreversible once the `pg_upgrade` command is executed.

The Zalando Postgres Operator primarily focuses on upgrades but may involve significant downtime. The search for a more flexible solution, especially with downgrade support and reduced downtime, remains ongoing.

## Conventional Strategy

The conventional strategy involves the following steps:

1. Stop the service to halt user requests.
2. Back up the database to be upgraded.
3. Set up the target version database and migrate the backup data.
4. Verify data consistency between the original and target databases.
5. If verification succeeds, resume the service and accept user requests.

{{< custom-img src="https://github.com/user-attachments/assets/500388a6-837c-4565-9b8f-277176392d73" cap="Conventional Strategy" >}}


While this approach effectively handles database version changes, it requires system downtime, which depends on the speed of data migration and validation—the faster these processes, the shorter the downtime.

However, modern businesses increasingly demand high availability alongside system stability. For example, achieving "five nines" (99.999% uptime) requires minimal downtime, as shown in table below. The conventional strategy falls short of meeting these strict requirements, highlighting the need for alternative solutions that mitigate downtime and maintain high availability during database upgrades.

| **Availability %** | **Downtime per year** | **Downtime per day (24 hours)** |
|--------------------|-----------------------|---------------------------------|
| 99%                | 3.65 days             | 14.40 minutes                  |
| 99.99%             | 52.6 minutes          | 8.64 seconds                   |
| 99.999%            | 5.26 minutes          | 864 milliseconds               |

## Our Solution

To address the challenges of database version changes and meet system design goals, the blue-green deployment strategy offers a practical solution.

The blue-green deployment strategy uses two parallel environments, "blue" and "green". Each environment hosts a different database version, enabling smooth transitions between versions with near-zero downtime. The key benefit is continuous system operation during the version change, significantly reducing downtime.

The strategy involves four main steps:

* Preparation: Set up both environments—blue hosting the current version and green the target version. The green environment is tested to ensure readiness for the transition.

* Data Synchronization: Synchronize data between the blue and green environments to ensure consistency. This process is automated, reducing manual effort and enhancing efficiency and reliability.

* Switching: After synchronization, the system switches from the blue to the green environment seamlessly, ensuring no service disruption, thus maintaining client transparency.

* Monitoring: Post-switch, the new database version is monitored for performance and consistency. If issues arise, the system can quickly revert to the blue environment to minimize client impact.

By using the blue-green deployment strategy, we can efficiently manage database version changes, automate processes, and maintain client transparency, overcoming the limitations of conventional methods and better supporting customers' needs.

### Achieving synchronisation between the Two PostgreSQL Databases


...

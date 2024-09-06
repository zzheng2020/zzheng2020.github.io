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
TocOpen: true
cover:
  image: "https://github.com/user-attachments/assets/ee6aca7f-a52f-4602-a3e1-4d2fbcfb24bb"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
---

This blog introduces my master's thesis, which is available [here](https://www.diva-portal.org/smash/record.jsf?aq2=%5B%5B%5D%5D&c=1&af=%5B%5D&searchType=UNDERGRADUATE&sortOrder2=title_sort_asc&language=en&pid=diva2%3A1781462&aq=%5B%5B%7B%22author%22%3A%5B%22Zhang%2C+Ziheng%22%5D%7D%5D%5D&sf=all&aqe=%5B%5D&sortOrder=author_sort_asc&onlyFullText=false&noOfRows=50&dswid=733) and the demo video can be found below. Most of the sections correspond to paragraphs from the thesis, while some parts that are necessary in the thesis but not needed in the blog have been omitted. This allows you to focus more on the project itself.

{{< youtube L9WOx1jLnF0 >}}

## Background

In any infrastructure, be it clustered or otherwise, databases often face challenges with version changes, including upgrades and downgrades. Upgrades are straightforward, driven by the need to adopt new features or improvements. However, downgrades, especially in the telecommunications industry, present unique challenges. For example, in the Access and Mobility Management Function (AMF) as shown in the image below, when a database is upgraded and data is converted to a new version, complications may arise. According to ETSI MANO specifications, a rollback to the previous version may be required in such cases.

{{< custom-img src="https://github.com/user-attachments/assets/9d7f0aa5-16b3-4da7-a8f6-1c4880c3ec6a" >}}

Another scenario is the "observation period" following an upgrade. For example, after upgrading a service from version A to version B, its performance is monitored. Key questions arise: Is the service functioning correctly? Is it handling the expected load? If the service underperforms, a rollback may be required, presenting the same challenges as in the first scenario. Both cases highlight the need for better support for database downgrades to mitigate service disruptions and maintain user experience.

...

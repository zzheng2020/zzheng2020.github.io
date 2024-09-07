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

This blog introduces my master's thesis ([[pdf]](https://www.diva-portal.org/smash/record.jsf?aq2=%5B%5B%5D%5D&c=1&af=%5B%5D&searchType=UNDERGRADUATE&sortOrder2=title_sort_asc&language=en&pid=diva2%3A1781462&aq=%5B%5B%7B%22author%22%3A%5B%22Zhang%2C+Ziheng%22%5D%7D%5D%5D&sf=all&aqe=%5B%5D&sortOrder=author_sort_asc&onlyFullText=false&noOfRows=50&dswid=733) [[presentation]](https://drive.google.com/file/d/1JTO0_wDDCrYbJaCS6PHPoJIcsNHOAGO2/view?usp=sharing) [[code]](https://github.com/zzheng2020/Master-Thesis)) and the demo video can be found below. Most of the sections correspond to paragraphs from the thesis, while some parts that are necessary in the thesis but not needed in the blog have been omitted. This allows you to focus more on the project itself.

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

The strategy involves four main steps as shown in the below image:

* Preparation: Set up both environments—blue hosting the current version and green the target version. The green environment is tested to ensure readiness for the transition.

* Data Synchronization: Synchronize data between the blue and green environments to ensure consistency. This process is automated, reducing manual effort and enhancing efficiency and reliability.

* Switching: After synchronization, the system switches from the blue to the green environment seamlessly, ensuring no service disruption, thus maintaining client transparency.

* Monitoring: Post-switch, the new database version is monitored for performance and consistency. If issues arise, the system can quickly revert to the blue environment to minimize client impact.

{{< custom-img src="https://github.com/user-attachments/assets/2b76a02f-1c26-417f-8699-fd7e6f2cb45d" cap="Blue-Green Strategy" >}}

By using the blue-green deployment strategy, we can efficiently manage database version changes, automate processes, and maintain client transparency, overcoming the limitations of conventional methods and better supporting customers' needs.

### Achieving Synchronisation between The Two PostgreSQL Databases

In the blue-green deployment strategy, synchronizing the two PostgreSQL databases is essential for maintaining data consistency and ensuring client transparency during version changes. We use logical replication as the primary method for database synchronization. It allows for selective, real-time data replication from one database to another. It ensures that changes in the source database are immediately reflected in the target database. This method is ideal for blue-green deployments, enabling smooth transitions between database versions with near-zero downtime and client disruption.

### Monitoring Synchronisation Progress

A critical factor in ensuring a smooth database version change is monitoring the synchronization progress between the master and follower nodes. In PostgreSQL, the `pg_stat_replication` view provides key replication status metrics to track synchronization completion.

The following metrics from the `pg_stat_replication` view are essential for monitoring synchronization progress:

* `sent_lsn`: Represents the latest Write-Ahead Log (WAL) position sent by the master to the follower.
* `pg_current_wal_flush_lsn`: Returns the latest WAL position flushed to disk on the master.
* `pg_wal_lsn_diff(lsn1, lsn2)`: Calculates the byte difference between two WAL positions.

#### Determining Synchronisation Completion

To determine when synchronization is complete, we compare the `sent_lsn` value with the `pg_current_wal_flush_lsn` value. When they match, it indicates that the latest WAL position sent by the master has been flushed to disk, marking the completion of synchronization.

Additionally, the `pg_wal_lsn_diff` function can track the remaining byte difference between the master and follower WAL positions, helping estimate the remaining synchronization time for more accurate system adjustments.

By utilizing the `pg_stat_replication` view and associated functions, we can effectively monitor synchronization progress, ensuring near-zero downtime, automation, and transparent operation during database version changes.

In practice, since clients may continuously write to the database, `sent_lsn` may never fully catch up with `pg_current_wal_flush_lsn`. However, the difference is typically small. To ensure synchronization, we temporarily halt write operations and wait for the values to match. This is why we refer to it as "near-zero" downtime rather than "zero" downtime.

### Integrating Blue-Green Deployment Strategy with Kubernetes

Custom Resource Definitions (CRDs) extend Kubernetes API, allowing users to define and manage custom resources.

To design the CRD, we define a schema with the following key fields:

* `apiVersion`: The API version used for the CRD.
* `kind`: The type of Kubernetes object, in this case, a custom resource for blue-green deployment.
* `metadata`: Includes details like name, namespace, labels, and annotations.
* `spec`: Specifies the properties of the custom resource.

## Implementation

We begin by discussing the database synchronization process and the configuration of master and follower nodes. The section then introduces the `PgUpgradeReconciler` structure and its integration with Kubernetes. The core of the implementation is the reconciliation process, which includes controller setup, deployment creation, service creation, schema synchronization, subscription management, Nginx proxy updates, resource cleanup, and supporting functions. Each step is carefully detailed to ensure smooth integration of the Blue-Green deployment strategy with Kubernetes, enabling database version changes with minimal downtime. This section serves as a practical guide to implementing this solution in real-world scenarios.

### Synchronisation between Databases

Both the master and follower nodes require specific configurations.

#### Master Node Configuration

The Kubernetes Operator automates these tasks to set up logical replication on the master node:

1. Configure PostgreSQL: Updates `postgresql.conf` to enable logical replication by setting `wal_level` to logical.
2. Create Publication: Uses the SQL command `CREATE PUBLICATION` to create a publication that includes all tables.
3. Export Schema: Runs the `pg_dump` command to export the database schema to an SQL file.

#### Follower Node Configuration

The Kubernetes Operator automates schema synchronization and subscription creation on the follower node:

1. Synchronize Schema: Imports the schema from the master using the `psql` command.
2. Create Subscription: Uses the SQL command `CREATE SUBSCRIPTION` to establish a subscription to the master node’s publication.

### `PgUpgradeReconciler` Structure

The `PgUpgradeReconciler` struct includes the following fields:

1. `client.Client`: Manages CRUD operations on Kubernetes objects.
2. Scheme: A `runtime.Scheme` for converting between Go structs and GroupVersionKinds.
3. Log: A `logr.Logger` instance for logging messages at various verbosity levels.

### Kubernetes RBAC

Kubebuilder annotations define the required RBAC permissions for the controller to manage resources:

1. `pgupgrades`: Custom resource for managing PostgreSQL upgrades.
2. `pgupgrades/status`: Subresource for handling `PgUpgrade` object statuses.
3. `pgupgrades/finalizers`: Subresource for managing finalizers on `PgUpgrade` objects.
4. `pods`, `deployments`, `services`, `configmaps`: Standard Kubernetes resources for creating, managing, and deleting related resources.

### Reconciliation Process

The `Reconcile` function is the heart of the controller, managing the reconciliation process. It starts by retrieving the `PgUpgrade` instance. If the instance is not found, it returns a nil error, allowing the controller to proceed with other instances.

#### Controller Setup

The `PgUpgradeReconciler` struct defines the controller for the `PgUpgrade` system. The `SetupWithManager` function sets up the controller with the Manager, associating it with the `pgupgradev1.PgUpgrade` resource.

#### Deployment Creation

The function checks if the `PgUpgrade` deployment exists. If not, it creates one using the `deploymentForPgUpgrade` function and logs the process. If it exists, it checks if the image matches the one in the `PgUpgrade` object and updates it if needed.

#### Service Creation

The function checks if the `PgUpgrade` service exists. If not, it creates one using the `serviceForPgUpgrade` function and logs it. If the service already exists, it proceeds.

#### Schema Synchronization

The `syncSchema` function runs schema synchronization on the target PostgreSQL instance by connecting to the target pod, retrieving its IP, and executing the schema sync command via the `remotecommand` package.

#### Subscription Creation

The `createSubscriptions` function sets up logical replication between the old and new PostgreSQL instances by connecting to the new instance and running the `CREATE SUBSCRIPTION` command with the necessary parameters.

#### Nginx Proxy Update

The `changeNginxProxyPass` function updates the Nginx configuration stored in a `ConfigMap`. It retrieves the `ConfigMap`, modifies the `proxy_pass` directive to point to the new follower’s IP, and updates the `ConfigMap` in the cluster.

#### Resource Deletion

The `deleteResource` function removes specified resources (e.g., deployments) from the cluster. It iterates over the resource names in the `KillDeployments` field of the `PgUpgrade` resource and deletes them, logging the process.

#### Helper Functions

Various helper functions support the system, including:

1. `labelsForPgUpgrade`: Generates labels for a `PgUpgrade` resource.
2. `getFollowerIP`: Retrieves the follower's IP from the cluster by fetching the pod's IP based on the `OldDBLabel`.

## Prototype Outcomes

The demo video above clearly shows the process and the outcome of this project. Here I would like to discuss the performance of our solution.

### Performance Analysis

We use `PgAdmin4` to analyze the performance metrics of the database within the Kubernetes cluster. These metrics include database sessions, transactions per second, tuples in/out, and block I/O.

Figure PA-1 shows the baseline database performance, with stable database sessions and tuples in. Transactions per second, tuples out, and block I/O follow a consistent and regular pattern.

{{< custom-img src="https://github.com/user-attachments/assets/660fe93e-9780-44dc-ba41-7cea7b5c7d11" cap="PA-1" >}}

Figure PA-2 depicts the system’s behavior after starting the client program, which runs five goroutines. Each goroutine reads from the database every three seconds and writes every ten seconds, reflecting the typical pattern of more frequent reads than writes. This activity causes a noticeable increase in all metrics, but the system remains stable.

{{< custom-img src="https://github.com/user-attachments/assets/72e3ddf5-6ac5-4012-a22d-70d1d293a575" cap="PA-2" >}}

Figure PA-3 illustrates the system’s response when our Kubernetes operator initiates a database version change. Aside from a minor increase in the "tuples in" metric (due to client operations), a brief fluctuation in metrics is observed, but the system quickly returns to normal. These rapid changes, highlighted by red boxes, are expected as the version change involves data synchronization between the master and follower nodes. Importantly, no idle sessions or transactions occur during this process, demonstrating that our solution achieves near-zero downtime while remaining transparent to clients.

{{< custom-img src="https://github.com/user-attachments/assets/e2a794ac-4463-4e13-b1cd-bb42aec19c5f" cap="PA-3" >}}

The results confirm that our solution effectively handles database version migrations and traffic switching. The smooth user experience during the migration highlights the approach's practicality and reliability.

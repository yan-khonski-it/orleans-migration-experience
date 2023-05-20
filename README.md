# orleans-migration-experience
Here, I put notes about Orleans Migration at Microsoft

# Big Picture

We have clients that want to run long tasks (3 - 10 minutes) that call external API and update data in the database. Each task has key.
Our services [web-api, silos] are deployed in multiple data-centers. At any moment of time, there could be multiple running tasks, but **for any key, there could be at most one running task at any time across data centers**.

> [More details] We need to update DNS topology of a service in multiple DNS providers / [traffic managers](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) and store resulted DNS configuration in database for given domain. 
> **Task key** is domain name for which we want to update topology.

## Reasons to upgrade the library
- We used old version of [Orleans 3.1.7](https://github.com/dotnet/orleans/tree/v3.1.7) which did not received updates, bug-fixes since end of May 2020.
- We received security alerts that we needed to upgrade libraries. Some old libraries could not be updated because that version of Orleans depended on them.
- We used .NET version 6, and we wanted to upgrade to [.NET 7](https://en.wikipedia.org/wiki/.NET) and further. We could not because old Orleans would not work there.
- We even had plans to get rid of Orleans and replace it with more robust and documented solutions, but first we needed to understand how to migrate.

# Terminology

[Actor model](https://en.wikipedia.org/wiki/Actor_model) Actor encapsulates state and behaviour. Actors receive messages, can send messages to other actors, create new actors, process message.

[Virtual actor](https://www.microsoft.com/en-us/research/publication/orleans-distributed-virtual-actors-for-programmability-and-scalability/?from=https://research.microsoft.com/apps/pubs/default.aspx?id=210931&type=exact) Assumes that actor always exists.
For example, you can have client code `actorFactory.getActor(actorId).performTask(inputArguments)`. Your client code can be running on machine 1, while while the factory will give you the the proxy representing the actor type.
However, the actual implementation of `performTask(inputArguments)` could be run on a different machine. More details further.

[Grain](https://learn.microsoft.com/en-us/dotnet/orleans/overview#what-are-grains) - virtual actor implementation in Orleans.

[Silo](https://learn.microsoft.com/en-us/dotnet/orleans/overview#what-are-silos) - application instance that manages grains. If there are multiple instances of silos running, those instance will agree on what silo instance hosts specific grains.
The client code `grainFactory.getGrain<GrainInterfaceType>(grainId)` will figure our on which silo the grain is hosted, and returns an object proxy to that grain.
When calling the actual method of the grain, the client will receive a promis, and the actual grain code will be executed in the silo hosting the grain.
https://learn.microsoft.com/en-us/dotnet/orleans/tutorials-and-samples/overview-helloworld

Clustering - silos need to agree on who is alive and active (so they can agree on grain hosting).

- https://learn.microsoft.com/en-us/dotnet/orleans/implementation/cluster-management
- https://learn.microsoft.com/en-us/shows/on-net/clustering-in-orleans
- https://learn.microsoft.com/en-us/dotnet/orleans/grains/grain-placement

Multi-clustering - similar to clustering, but across multiple datacenters. This actually ensures that at any moment of time, at most one task with given key can be run.
- https://learn.microsoft.com/en-us/dotnet/orleans/deployment/multi-cluster-support/overview

# Dependencies

## CosmosDB
We use [Azure ComsosDB](https://learn.microsoft.com/en-US/azure/cosmos-db/introduction) for storing the grain state. In our case, grain represented domain topology configuration, where grain id (actor key) is domain name.

## Azure Table Storage
We used [Azure Storage Table](https://learn.microsoft.com/en-us/azure/storage/tables/table-storage-overview) so silos can communicate about statuses of themselves and neighbours.
>  First, it is used as a rendezvous point for silos to find each other and Orleans clients to find silos. Second, we use reliable storage to help us coordinate the agreement on the membership view.

https://learn.microsoft.com/en-us/dotnet/orleans/implementation/cluster-management

Also, we use it for Multi-clustering - silos in different clusters should discover each other.
> Once a silo has learned and cached the location of a grain activation (no matter in what cluster), it sends messages to that silo directly, even if the silo is not a cluster gateway (in a separate cluster).

https://learn.microsoft.com/en-us/dotnet/orleans/deployment/multi-cluster-support/gossip-channels

## Azure Key Vault
We stored our configuration (both property names and secrets) inside [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview).

# Implementation details

## Plan. 
At the beginning I could not estimate the task because I wasnot familiar with most of the tach stack. I started with learnign and reading documentation.

### Backup and restore
In order to proceed with the migration, I needed to prepare for the worst.
[Back up and restore for CosmosDB](https://learn.microsoft.com/en-us/azure/cosmos-db/restore-account-continuous-backup).
[Azure Table storage did not have recovery](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-storage/data-protection-backup-recovery#non-supported-storage-recovery), but I found a manual work around with
[Azure Storage Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer)
Later, I found out that we do not need to restore Azure Table Storage, we can delete the table completely, and it will be populated automatically after some time.
Learned how to deploy a new version of our services and rollback to the previous version. (Fortunately, I did not need to use it, but it is importan to know this in case of a problem).

// TODO Update

Created a plan for migration, including back up strategy for CosmosDB and Azure Storage Table. Verified that our test and prod environments have the same configuration for clustering (services need to schedule jobs in the cluster, but at most one job with given key is allowed to run at any moment) and multi-clustering (services should agree about scheduling jobs, but at most one job with given key is allowed to run across multiple datacenters). Microsoft removed multi-clustering support from Orleans. Discovered and used a fork (created internally at Microsoft) of Orleans with multi-clustering support.  I studied four  repositories [our code, the fork code, original Orleans code, and another project that used the fork]. Re-wrote configuration related code (some classes were removed had to implementing missing parts). New version of Olreans changed id format of jobs state stored in CosmosDB. I fix the logic responsible for id generation, added tests for enforcing old id format is used. Finally, I found and fixed issues related that some services in different datacenters could not discover each other (potentially multiple jobs with the same key could run and corrupt the data). Deployed the migration, very short downtime (one region was not accepting the traffic, deployed there, deployed into the active region)  



Created plan and at the end tested that migration did not corrupt the data and multi-clustering and clustering conditions are satisfied. 
whiSolved problems with clustering (silo instances should be discovered in one datacenter) and multi-clustering (services should be discovered in different datacenters and be able to distributed the work). The problem was that official Microsoft.Orleans removed support for multi-clustering.
https://github.com/dotnet/orleans/issues/7485
https://learn.microsoft.com/en-us/dotnet/orleans/migration/
https://github.com/dotnet/orleans/commit/ff7b602f5be4bfde9f5c80e77462d327d2c20bef
It enabled us to upgrade other libraries.


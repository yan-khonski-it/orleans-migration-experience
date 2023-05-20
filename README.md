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

Silo gateway - TODO add definition

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

## Azure VNET
We use VNets to connect multiple deployments within a region, and gateways to connect VNets across different regions.
https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview

## Azure Key Vault
We stored our configuration (both property names and secrets) inside [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview).

# Implementation details
## Pre plan
I tried to upgrade Orleans into the next version [3.2.0](https://github.com/dotnet/orleans/releases/tag/v3.2.0). It is the next version after 3.1.7. However, it had breaking change, [it removed multiclustering support](https://github.com/dotnet/orleans/pull/6498).
Some classes were removed, so our code did not compile. I had to revert the changes and make proper plan.

## Asking why
- *Why did we use Orleans?*
For running long tasks, making sure than no one else could modify the domain configuration until the task is finished.

- *Why did we need clustering?*
Orleans required clustering for silos to agree on hosting grains.

- *Why did we need multi-clustering*?
We wanted our service to be highly available, so we run it in multiple data cetners. However, since the grain state is stored in CosmosDB that allows [multi-region writes](https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-multi-master?tabs=api-async); 
we wanted to make sure that at most one task for given domain can be running at any moment of time, no one else should modify that domain configuration.

- *Could we replace or avoid multi-clustering*?
It is our next goal after the Orleans migration is done. There were several ideas, I just mention them briefly here.
They require futher research and time, and our team had to focus on other tasks, while I was working on the migration.
    - Allow only a single region to write data, while the other region is stand by.
    - Implement custom locking mechanism.
    - Use messaging, for example, [Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/compare-messaging-services#azure-service-bus), and make sure that a message can be consumed by one consumer.
We did not want to use Kafka, because it would bring more operational work (we wanted to use existing tools, but we may change our mind in the future though).
We did not want to use "high level" Azure Services such as Service Bus, because if such service goes down, the whole Microsoft Identity Organization could also go down. We wanted to use "Low level" services that would not affect Identity Services.

At the end I had to document all these things into our team Wiki.

## Plan. 
At the beginning I could not estimate the task because I was not familiar with most of the tach stack. I started with learnign and reading documentation.
Prepared questions for myself, answred them, documented in Wiki.
At this moment I already new that using official Microsoft Orleans that removed multi-clustering was not an option for us.
I had one option - to implemnent multi-clustering myself. Here some ideas how it could be done.
https://github.com/dotnet/orleans/issues/7485

However, after I have talked to several teams that use official Microsoft.Orleans I found one team that also needed multi-clustering, so they have already implemented it.
That team, let me name it DFP, used their fork of Orleans, which they named DFP.Orleans. 

Now, I had to decide if we could use the fork DFP.Orleans (this fork is in Microsoft private repository, so I cannot share details about it).
The idea they used, they require that silos in multiple regions should use the shared VNET.
https://learn.microsoft.com/en-us/dotnet/orleans/deployment/multi-cluster-support/gossip-channels

### Backup and restore
In order to proceed with the migration, I needed to prepare for the worst.
[Back up and restore for CosmosDB](https://learn.microsoft.com/en-us/azure/cosmos-db/restore-account-continuous-backup).
[Azure Table storage did not have recovery](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-storage/data-protection-backup-recovery#non-supported-storage-recovery), but I found a manual work around with
[Azure Storage Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer)
Later, I found out that we do not need to restore Azure Table Storage, we can delete the table completely, and it will be populated automatically after some time.
Learned how to deploy a new version of our services and rollback to the previous version. (Fortunately, I did not need to use it, but it is importan to know this in case of a problem).





-------------------------------------------------
// TODO Update

Created a plan for migration, including back up strategy for CosmosDB and Azure Storage Table. Verified that our test and prod environments have the same configuration for clustering (services need to schedule jobs in the cluster, but at most one job with given key is allowed to run at any moment) and multi-clustering (services should agree about scheduling jobs, but at most one job with given key is allowed to run across multiple datacenters). Microsoft removed multi-clustering support from Orleans. Discovered and used a fork (created internally at Microsoft) of Orleans with multi-clustering support.  I studied four  repositories [our code, the fork code, original Orleans code, and another project that used the fork]. Re-wrote configuration related code (some classes were removed had to implementing missing parts). New version of Olreans changed id format of jobs state stored in CosmosDB. I fix the logic responsible for id generation, added tests for enforcing old id format is used. Finally, I found and fixed issues related that some services in different datacenters could not discover each other (potentially multiple jobs with the same key could run and corrupt the data). Deployed the migration, very short downtime (one region was not accepting the traffic, deployed there, deployed into the active region)  



Created plan and at the end tested that migration did not corrupt the data and multi-clustering and clustering conditions are satisfied. 
whiSolved problems with clustering (silo instances should be discovered in one datacenter) and multi-clustering (services should be discovered in different datacenters and be able to distributed the work). The problem was that official Microsoft.Orleans removed support for multi-clustering.
https://github.com/dotnet/orleans/issues/7485
https://learn.microsoft.com/en-us/dotnet/orleans/migration/
https://github.com/dotnet/orleans/commit/ff7b602f5be4bfde9f5c80e77462d327d2c20bef
It enabled us to upgrade other libraries.


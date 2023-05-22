# orleans upgrade experience
Here, I put notes about Orleans upgrade - a task that I finished at Microsoft

# Big Picture

We have clients that want to run long tasks (3 - 10 minutes) that call external API (that could be retried) and update data in the database. Each task has a unique key.
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
However, the actual implementation of `performTask(inputArguments)` could be run on a different machine (similar to RMI). More details further.

[Grain](https://learn.microsoft.com/en-us/dotnet/orleans/overview#what-are-grains) - virtual actor implementation in Orleans.

[Silo](https://learn.microsoft.com/en-us/dotnet/orleans/overview#what-are-silos) - application instance that manages grains. If there are multiple instances of silos running, these instance will agree on what silo instance hosts specific grains.
The client code `grainFactory.getGrain<GrainInterfaceType>(grainId)` will figure our on which silo the grain is hosted, and returns an object proxy to that grain.
When calling the actual method of the grain, the client will receive a promise, and the actual grain code will be executed in the silo hosting the grain.
https://learn.microsoft.com/en-us/dotnet/orleans/tutorials-and-samples/overview-helloworld

Cluster gateway - a subset of active silos in the given cluster, that advertise their IP address. When silos in different clusters are discovering each other, silo gateway works as "points of first contact".
> Once a silo has learned and cached the location of a grain activation (no matter in what cluster), it sends messages to that silo directly, even if the silo is not a cluster gateway.

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

Also, we use Azure Table Storage for Multi-clustering - silos in different clusters should discover each other.

https://learn.microsoft.com/en-us/dotnet/orleans/deployment/multi-cluster-support/gossip-channels

## Azure VNET
We use VNets to connect multiple deployments within a region, and we also use VNET to connect gateways across different regions.
https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview

## Azure Key Vault
We store our configuration (both property names and secrets) inside [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview).

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

Now, I had to decide if we could use the fork DFP.Orleans (this fork is in Microsoft private repository, so I cannot put links here).
The idea they used, they require that silos in multiple regions should use the shared VNET.
https://learn.microsoft.com/en-us/dotnet/orleans/deployment/multi-cluster-support/gossip-channels

Then  they implemented multi-clustering similar way as they had for clustering.

## Backup and restore
In order to proceed with the migration, I needed to prepare for the worst.
[Back up and restore for CosmosDB](https://learn.microsoft.com/en-us/azure/cosmos-db/restore-account-continuous-backup).
[Azure Table storage did not have recovery](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-storage/data-protection-backup-recovery#non-supported-storage-recovery), but I found a manual work around with
[Azure Storage Explorer](https://azure.microsoft.com/en-us/products/storage/storage-explorer)
Later, I found out that we do not need to restore Azure Table Storage, we can delete the table completely, and it will be populated automatically after some time.
Learned how to deploy a new version of our services and rollback to the previous version. (Fortunately, I did not need to use it, but it is importan to know this in case of a problem).

## Code details
There we no documentation, so I had to debug our code with old Orleans and compare it with DFP.Orleans and this is how I fixed issues.

A few things I had to change: 
- Remove all code that depended on `MultiClusterOptions` because this class was removed. Re-write such components, so they use `GlobalClustering` configuration.
- Replace deprecated methods with the methods that had similar result. In my case, I had to tell Orleans how to connect to Azure Table Storage and use it for clustering.

I found a new error in logs (saying that classes of grains impmenenting [`IGrainWithStringKey`](https://learn.microsoft.com/en-us/dotnet/api/orleans.igrainwithstringkey?view=orleans-7.0) were not properly configured, so their grains cannot be found).
This error was misleading, after debugging I found out that this log message was old code that DFP team forgot to change. I verified it with their team.

Also, I had a problem with grain id. Unofrtunately, people who wrote the logic, decided that grain id was generated by the Orleans. our code was similar to 
```
String grainId = grain.getId().ToString();
grainStateRepository.store(grainId, grain);
```

I found out that the new version of Orleans changed ID format for grains, therefore, we could not read old grains from the database.
src/Orleans.Core.Abstractions/IDs/GrainId.cs
commit
https://github.com/dotnet/orleans/commit/0894246444c7101a54905a0f0a402bfc27f1054e#diff-6ac8d163b8c760895ac0b8b0977e9d00b298e6ca6cef411786bd57cd528f3888

To fix the issue,  I had to copy the logic of id generation from the old code. Fortnately, I made work-around code short and clear - I used reflection API to retrieve the fields required for ID generation and passed those fields into the original logic for id generation.

Additionally, I added tests for all our grains to make sure that our grain id will not change.
As an alternative, I could run a database migration where I would read grains with old id and re-write them with new id, but that would have taken longer.

After testing [more details bellow], I found out that multi-clustering was not working. Silos discovered each other in the same cluster, but they could not talk to the silos in different cluster.
I had to debug two code bases: our project and DFP project to compare the configuration. After checking git history, I found out that DFP team fixed one bug related to running applications on local machine, so I had to enabled two of feature flags, that were disabled by default and not documented.

After each fix and iteration, I had to test.

## How I tested



-------------------------------------------------
// TODO Update

Created a plan for migration, including back up strategy for CosmosDB and Azure Storage Table. Verified that our test and prod environments have the same configuration for clustering (services need to schedule jobs in the cluster, but at most one job with given key is allowed to run at any moment) and multi-clustering (services should agree about scheduling jobs, but at most one job with given key is allowed to run across multiple datacenters). Microsoft removed multi-clustering support from Orleans. Discovered and used a fork (created internally at Microsoft) of Orleans with multi-clustering support.  I studied four  repositories [our code, the fork code, original Orleans code, and another project that used the fork]. Re-wrote configuration related code (some classes were removed had to implementing missing parts). New version of Olreans changed id format of jobs state stored in CosmosDB. I fix the logic responsible for id generation, added tests for enforcing old id format is used. Finally, I found and fixed issues related that some services in different datacenters could not discover each other (potentially multiple jobs with the same key could run and corrupt the data). Deployed the migration, very short downtime (one region was not accepting the traffic, deployed there, deployed into the active region)  



Created plan and at the end tested that migration did not corrupt the data and multi-clustering and clustering conditions are satisfied. 
whiSolved problems with clustering (silo instances should be discovered in one datacenter) and multi-clustering (services should be discovered in different datacenters and be able to distributed the work). The problem was that official Microsoft.Orleans removed support for multi-clustering.
https://github.com/dotnet/orleans/issues/7485
https://learn.microsoft.com/en-us/dotnet/orleans/migration/
https://github.com/dotnet/orleans/commit/ff7b602f5be4bfde9f5c80e77462d327d2c20bef
It enabled us to upgrade other libraries.


# orleans-migration-experience
Here, I put notes about Orleans Migration at Microsoft

# Big Picture

We have clients that want to run long tasks (3 - 10 minutes) that call external API and update data in the database. Each task has key.
Our services [web-api, silos] are deployed in multiple data-centers. At any moment of time, there could be multiple running tasks, but **for any key, there could be at most one running task at any time across data centers**.

> [More details] We need to update DNS topology of a service in multiple DNS providers / [traffic managers](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) and store resulted DNS configuration in database for given domain. 
> **Task key** is domain name for which we want to update topology.

## Reasons to upgrade the library
- We used old version of [Orleans 3.1.7](https://github.com/dotnet/orleans/tree/v3.1.7) which did not received updates since end of May 2020.
- We received security alerts that we needed to upgrade libraries. Some old libraries could not be updated because that version of Orleans depended on them.
- Old version of Orleans did not receive updates and bug fixes.
- We used .NET version 6, and we wanted to upgrade to [.NET 7](https://en.wikipedia.org/wiki/.NET) and further. We could not because old Orleans would not work there.
- We even had plans to get rid of Orleans and replace it with more robust and documented solutions, but first we needed to understand how to migrate.

# Terminology

[Actor model](https://en.wikipedia.org/wiki/Actor_model) actors execute tasks. Actor encapsulates state and behaviour.

// TODO Update

Created a plan for migration, including back up strategy for CosmosDB and Azure Storage Table. Verified that our test and prod environments have the same configuration for clustering (services need to schedule jobs in the cluster, but at most one job with given key is allowed to run at any moment) and multi-clustering (services should agree about scheduling jobs, but at most one job with given key is allowed to run across multiple datacenters). Microsoft removed multi-clustering support from Orleans. Discovered and used a fork (created internally at Microsoft) of Orleans with multi-clustering support.  I studied four  repositories [our code, the fork code, original Orleans code, and another project that used the fork]. Re-wrote configuration related code (some classes were removed had to implementing missing parts). New version of Olreans changed id format of jobs state stored in CosmosDB. I fix the logic responsible for id generation, added tests for enforcing old id format is used. Finally, I found and fixed issues related that some services in different datacenters could not discover each other (potentially multiple jobs with the same key could run and corrupt the data). Deployed the migration, very short downtime (one region was not accepting the traffic, deployed there, deployed into the active region)  



Created plan and at the end tested that migration did not corrupt the data and multi-clustering and clustering conditions are satisfied. 
whiSolved problems with clustering (silo instances should be discovered in one datacenter) and multi-clustering (services should be discovered in different datacenters and be able to distributed the work). The problem was that official Microsoft.Orleans removed support for multi-clustering.
https://github.com/dotnet/orleans/issues/7485
https://learn.microsoft.com/en-us/dotnet/orleans/migration/
https://github.com/dotnet/orleans/commit/ff7b602f5be4bfde9f5c80e77462d327d2c20bef
It enabled us to upgrade other libraries.


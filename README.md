# orleans-migration-experience
Here, I put notes about Orleans Migration at Microsoft


Created a plan for migration, including back up strategy for CosmosDB and Azure Storage Table. Verified that our test and prod environments have the same configuration for clustering (services need to schedule jobs in the cluster, but at most one job with given key is allowed to run at any moment) and multi-clustering (services should agree about scheduling jobs, but at most one job with given key is allowed to run across multiple datacenters). Microsoft removed multi-clustering support from Orleans. Discovered and used a fork (created internally at Microsoft) of Orleans with multi-clustering support.  I studied four  repositories [our code, the fork code, original Orleans code, and another project that used the fork]. Re-wrote configuration related code (some classes were removed had to implementing missing parts). New version of Olreans changed id format of jobs state stored in CosmosDB. I fix the logic responsible for id generation, added tests for enforcing old id format is used. Finally, I found and fixed issues related that some services in different datacenters could not discover each other (potentially multiple jobs with the same key could run and corrupt the data). Deployed the migration, very short downtime (one region was not accepting the traffic, deployed there, deployed into the active region)  

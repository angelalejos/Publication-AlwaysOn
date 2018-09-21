# Publication-AlwaysOn
What is Supported?

1) SQL Server replication supports the automatic failover of the publisher, the automatic failover of transactional subscribers, and the manual failover of merge subscribers. The failover of a distributor on an availability database is not supported.

2) In an AlwaysOn availability group a secondary database cannot be a publisher. Re-publishing is not supported when replication is combined with AlwaysOn Availability Groups.

 
Environment

AlwaysOn

SRV1: Synchronous Replica          - Current Primary

SRV2: Synchronous Replica

SRV3: Asynchronous Replica

Availability Group  :MyAvailabilityGroup

AG database        : MyNorthWind

AG Listener          : AGListener



 

 

Below is the environment we will be building at the end of this blog:

SRV1: Original Publisher

SRV2: Publisher Replica

SRV3: Publisher Replica

SRV4: Distributor and Subscriber (You can choose a completely new server to be the distributor as well, however do not have a distributor on any of the publishers in this case as the failover of a distributor is not supported in this case). 

Overview

The following sections build the environment described above:

- Configure a remote distributor

- Configure the Publisher at the original Publisher

- Configure Remote distribution on possible publishers

- Configure the Secondary Replica Hosts as Replication Publishers

- Redirect the Original Publisher to the AG Listener Name

- Run the Replication Validation Stored Procedure to verify the Configuration

- Create a Subscription

 

1. Configure a remote distributor

The distributor should not be on the current (or intended) replica of the availability group of which the publishing database is part of. This just means that Distributor in our case, should not be on SRV1, SRV 2, SRV3 because these servers are part of the AG that has the publishing database (MyNorthWind).

We can have a dedicated server (which is not part of the AG) acting as a distributor or we can have the distributor on the subscriber (provided subscriber is not part of an AG).

 Let's configure distribution on SRV4.

Right click on Replication and select "Configure Distribution". 


 

 

We'll select the first option as we want SRV4 as a distributor.



 

 Specify the snapshot folder location.


 

 

We'll go with the default distribution database folder.
 



 

 

In the below screen, we need to specify SRV1, SRV2 and SRV3 as publishers. Click on Add and then Add SQL Server Publisher. Connect to the 3 servers that can act as publishers. Note that SRV4 already exists in the list and you can choose to leave it that way.
.

 

 

This is how it should look like with all publishers added.


 

 

Enter in the password that the remote publishers will use to connect to the distributor.


 

 

Click on Next and then "Configure Distribution" and then next.
Click on Finish and now, the distribution is successfully set up.
 

2. Configure the Primary Replica as the original Publisher

Define SRV1 as the original publisher as it is currently the primary replica. You can have any of the AG replica as the original publisher, as long as it is the current primary replica.

 

In SQL Server Management Studio, use Object Explorer to connect to SRV1 and drill into Replication and then Local Publications.


 

Right click Local Publications and choose New Publication. Click Next.
In the Distributor dialog, choose the option 'Use the following server as the Distributor', click the Add button and add SRV4. Click Next.
Enter the same password that was used in Step 7 of "Configure the distributor".


 

 

Select the database to be published: MyNorthWind


 

 We'll be setting up Transactional Publication.


 

 

In the Articles and Filter Table rows dialogs, make your selections.
In the Snapshot dialog, for now, choose the 'Create a snapshot immediately..' and click Next.
In the Agent Security dialog box, specify the account under which Snapshot Agent and Log Reader Agent will run. You can also use the SQL Server Agent account to run the Snapshot Agent and Log Reader Agent.
In the Wizard Actions dialog, select 'Create the publication' and click Next.
Give the publication a name and click Finish in the Complete the Wizard dialog.


 

 

3) Configure Remote distribution on possible publishers

For the possible publishers and secondary replicas: SRV2 and SRV3, we'll have to configure the distribution as a remote distribution that we created on SRV1.

Launch SQL Server Management Studio. Using Object Explorer, connect to SRV2 and right click the Replication tab and choose Configure Distribution. Choose 'Use the following server as the Distributor' and click Add. Select SRV4 as the distributor.
 





 

In the Administrator Password dialog, specify the same password to connect to the Distributor.


 



In the Wizard Actions dialog, accept the default and click Finish.
Click finish and follow the same steps on SRV3 to configure the distribution as SRV4.
 

4) Configure the Secondary Replica Hosts as Replication Publishers

 In the event that a secondary replica transitions to the primary role, it must be configured so that the secondary can take over after a failover. All possible publishers will connect to the subscriber using a linked server. To create a linked server to the subscriber, SRV4 , run the below query on the possible publishers: SRV2 and SRV3.

EXEC sys.sp_addlinkedserver

@server = 'SRV4';

 

5) Redirect the Original Publisher to the AG Listener Name

We have already created an AG listener named AGListener. At the distributor (Connect to SRV4) , in the distribution database, run the stored procedure sp_redirect_publisher to associate the original publisher and the published database with the availability group listener name of the availability group.

 

USE distribution;

GO

EXEC sys.sp_redirect_publisher

@original_publisher = 'SRV1,

@publisher_db = 'MyNorthWind',

@redirected_publisher = 'AGListener';

 

6) Run the Replication Validation Stored Procedure to verify the Configuration

At the distributor (SRV4), in the distribution database, run the stored procedure sp_validate_replica_hosts_as_publishers to verify that all replica hosts are now configured to serve as publishers for the published database.

 

USE distribution;

GO

DECLARE @redirected_publisher sysname;

EXEC sys.sp_validate_replica_hosts_as_publishers

@original_publisher = 'SRV1',

@publisher_db = 'MyNorthWind',

@redirected_publisher = 'AGListener';

 

The stored procedure sp_validate_replica_hosts_as_publishers should be run from a login with sufficient authorization at each availability group replica host to query for information about the availability group. Unlike sp_validate_redirected_publisher, it uses the credentials of the caller and does not use the login retained in msdb.dbo.MSdistpublishers to connect to the availability group replicas.

 

7) Create a subscription

Right click on the publication: Publication_AlwaysOn and select New Subscriptions.


 

 Select the publication on SRV1.
 



 

We'll create a push Subscription, however a pull subscription will work as well.
 



 

Select the subscriber instance as SRV4 and a subscriber database
 

 



Select the SQL Server Agent credentials to run the Distribution Agent.
 



 



 

Select "Initialize at First Synchronisation" on the subscriber SRV4.
 



Select the subscriber instance as SRV4 and a subscriber database.


 

How to use the Replication Monitor ?

After failover to a secondary replica, Replication Monitor is unable to adjust the name of the publishing instance of SQL Server and will continue to display replication information under the name of the original primary instance of SQL Server. After failover, a tracer token cannot be entered by using the Replication Monitor, however a tracer token entered on the new publisher by using Transact-SQL, is visible in Replication Monitor.

At each availability group replica, add the original publisher to Replication Monitor.

 

Publisher failover Demonstration

In this section, we'll failover the Availability Group from the current primary replica and replication publisher: SRV1 to secondary replica and possible publisher:SRV2. This will not impact the working of Replication in any way.



 

The Failover Availability Group Wizard comes up.


 

Select the secondary replica you want to failover the AG to, in this case, SRV2.
 

 

Connect to SRV2 which is the SQL instance acting as the secondary replica.
 

 

Click on Finish and the failover to SRV2 should complete successfully.
We can also failover to asynchronous secondary replica and possible publisher, SRV3 in the same way.
 



 

This will cause data loss as SRV3 is an Asynchronous Replica.


 

Click on Finish.
However, after the failover to a Asynchronous secondary replica, the data movemnet on the AG database, MyNorthWinds is paused on the 2 secondary replicas-SRV1 and SRV2.
The database state will show "Not Synchronizing" on SRV2 and SRV1.
 

 

Right-click the availability database, MyNorthwind under AlwaysOn High Availability drop-down and select "Resume Data Movement". Follow the same on SRV1.
 

We can go with the default selection "Continue executing after error".
 

 

 

Resume data movemnet on SRV1 as well and AlwaysOn database MyNorthWind will show as "Synchronizing" instead of "Synchronized" as SRV3 is the primary replica now and it was et as an Asynchronous Replica initially.
After making these changes, Replication will function as usual.

Reference: https://blogs.msdn.microsoft.com/alwaysonpro/2014/01/30/setting-up-replication-on-a-database-that-is-part-of-an-alwayson-availability-group/

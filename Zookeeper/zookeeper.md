If you are related to application development you have probably heard of ZooKeeper at some point in time. Its a nice little project, and in fact it’s used in a lot of Apache projects. Its found in Kafka, SOLR, HBase and many more.

In very simple terms, ZooKeeper is a Co-ordination service. On reddit there is a nice subsection that's called `explain like I'm five` you ask questions and the responses are given in such simple terms that even a 5 year old would understand. I'll make an attempt to do the same : )

So if you were a 5 year old developer and you want to know what this ZooKeeper business is about and how to use it. Let's dive right in!

For any service
2 different kinds of systems want to co-ordinate with each other

Think a scoreboard on a basketball or football game. Its out there everyone can read the score and pickup the information. The scoreboard is the central information system where all the people(clients) can fetch and get the latest information from a central place. The team playing the game at the center plays the game and whatever they do like scoring goals and baskets writes information to this board. Its a score information co-ordination system.

Another example of co-ordination is a support call center phone number. Imagine there was no central phone number and people called these cellphone numbers of the agents, some of them may be busy with other customers, some may be off duty, others may have called in sick or may be on leave. As a customer it would be very frustrating to call different people with the chance they will be available to support you. If you are told that all customer request have to be services in the first call they make. The odds would be squarely against you, and there's so chance you'd be able to meet this objective.

Looking at it from a different perspective, basically the request for support comes in but the agents are serving the requests are not reliably available all through the 24 hours of the day and even across the entire month they are not available for days at a time. The customers can not call the agents directly.

There has to be something in between. What we need is a co-ordination service. A service where agents will register and make themselves available to take calls. Calls will be routed only to agents that can service calls. And for the customer side that makes the support call requests they shouldn't be calling tens and hundreds of different numbers. Call one number, and you will be routed to the agent that is free.  The agents will keep changing, they will not be available everyday for x number of hours, they may even leave and never return back to answering calls. The customer calling the line is not bothered with this churn and failures to be able to service calls. There only interface with the layer of abstraction that is the call center number. The number will give them access to an active agent that is available.

Cut back to your data center and your web app. We can say safely that the days of scaling an application by provisioning large dedicated machines and building them bigger and bigger are gone now.

These days most applications that are expected to scale are either on VMs or on some sort of a private of public cloud.  There is a philosophy to managing servers that says, Your servers should be like cattle, not pets. Which means, if one of them dies replace it with another one. If your capacity is low, add more machines and remove them if resources are unused. This is the horizontal approach to scaling. Combined with horizontal scaling there is also another approach to architecting applications by building Micro Services and using a Service Oriented Architecture. This would mean the various components of your application would be separate services that could be called individually. This would mean that if a certain part of the application failed it would be only that one feature of your application that fails and not the whole application. Here Netflix explains how it implements this internally.

In this context even your code needs to fetch data related to a certain feature, it needs to make a call to separate machines over your network to fetch the data. Now, with each service horizontally scaling up and down, you need to have some co-ordination happening between the requester(your code) and the machine that services your request for the data. Here is where ZooKeeper comes in.

ZooKeeper basically let you store small amounts of information within itself. This information can be queried by clients to get addresses of active servers.

As an example if you have a master database and 3 other slave databases set in replication to the master. You could read information from your slave databases to distribute the load away from the master database.

Typically you would have the ip addresses of these databases put in some config file somewhere and the code would expect the servers to be available all the time.

A different approach in this case would be, That the application does not have information about the database servers that will be used for reading data. What it does is, it goes to ZooKeeper and asks for the current database ip address for read purpose. ZooKeeper provides it this data and the query can be made.

With this simple abstraction, what you have done is decoupled the number of databases and the actual IP address from the application configuration / code. You can now dynamically manipulate this data. You can now do all these things that wouldn't be possible with the previous approach :
- Have your requests evenly distributed in a round-robin fashion. Similar benifits as a software load balancer.
- Scale horizontally to n number of instances, i.e. add more database servers when you load goes up without having to change any code or configuration on the application.
- Have a resilient architecture where nodes could fail and they would be proactively removed.
- Have the ability to do rolling maintenance activities on the servers without downtime.


ZooKeeper exposes a linux directory like structure where you can have paths that can be defined as a hierarchy. Only difference is, in ZooKeeper you can save data in the folder level also and on the last zNode level as well. ZooKeeper is normally deployed in a cluster because it can become a single point of failure, so it needs to be resilient itself as well. The thing to note in this setup is that the data in all the nodes remains exactly the same. Values that are stored in ZK are normally stored in the memory. This makes queries to ZK very fast. The general setup of a ZK clusters has either 3, 5, 7 or more nodes.

There is are a couple of other great options too in ZooKeeper that can be leveraged.

Ephemeral zNodes
Ephemeral stands for that which is not permanent. These zNodes only live till the session with ZK of the process that created it is active.

This means that if a script creates a zNode on ZooKeeper and sets it as ephemeral while creating it, as soon as the script is killed, the zNode that it created also dissapears.

Combined with the previous example, this feature can be used to track only active nodes that can service requests. On each node that provides a service, like a database server there would be a script that would run in the background which would connect to ZK, create an ephemeral node with details of the server and the service it provides. This script also keeps validating if the service it is advertising for is in an active state. If one of the following happens :
The database crashes, then the script realises this and removes the ephemeral node.
If the server itself crashes, then the script will also stop. This would also stop the session with ZK and thus the ephemeral node that it created would get deleted automatically.

The caveat here is that you need to ensure that you script keeps running and you have a cron-job of one minute that checks for and starts the script it its down. Otherwise your database could be up but your script could fail and this would mean your database would not be registered with ZK and it would not be reachable.




Watchers
Watchers lets you keep an eye for changes on a particular zNode. This means that you would get an event if any activity happens on that particular zNode. This would be a signal to your script that you need to pull the latest data from the zNode.  

The thing to note here is, the event is there to only let you know that you have old information and there is some new information on the zNode you are watching. It doesn't give you events everytime changes happen. It gives the notification only once until the next time you pull data from it. After your pulling data if something new happens again, it will give you a notification once.

Quorum
As we discussed earlier that ZK can work in a cluster of several machines. In a multi node setup of ZK there is a concept of a master node and slave nodes. The difference between them is that reads can happen from anywhere but writes will only happen from the master. They will then be synced to the slave servers. It is possible that even one of the nodes of ZK can crash. If the crashed server is a slave, it doesn't make a difference, but if the server is a master then the remaining machines would be left with no master. To take care of this situation the slaves would get together and conduct an election amongst themselves. The node with the oldest data would become the next Master.

If your application has some concept of Master and Slave where slaves need to be defined and the master could crash. You can use this feature of ZK to decide who should be the next Master inside your application as well.

This wraps up a really long explanation of ZK, hope it helps you. : )

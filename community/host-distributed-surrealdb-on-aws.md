---
title: Deploying a distributed SurrealDB cluster
creator: Bibaswan Bhawal
link: https://github.com/bibaswan-bhawal
---

SurrealDB is a very powerful database, but one of its greatest features is its abilty to be hosted as a cluster. This guide will walk you through deploying a distributed SurrealDB cluster on AWS. This is the method I used to deploy SurrealDB for my project.

A SurrealDB cluster has three main components:

- SurrealDB Client Cluster
- Placement Driver Cluster
- TiKV Node Cluster

Both the Placement Driver Cluster and the Client Cluster will sit behind a load balancer so that traffic can get routed automatically and in the case of the client cluster it can also be set up to scale automatically.

The setup of the TiKV cluster is slightly more complex as you have to use the TiUP command line tool to create, update, scale a TiKV cluster. Therefore, it has to be done manually. There are methods to automate this as well but that is beyond the scope of this tutorial.

<br />

# Prerequisite: Setting up AWS

The first thing we need to set up for a cluster is a [Virtual Private Cloud (VPC)](https://www.cloudflare.com/learning/cloud/what-is-a-virtual-private-cloud/) for our cluster. Essentially the idea is that we will be able to isolate which clusters can be accessed from public IPs and which can only be accessed from inside the VPC.

There are many tutorials out there to create a VPC in AWS and similar things can be achieved with different cloud providers.

The structure you want is as follows:

SurrealDB VPC:

- 2 - 4 Private Subnets in different availablity zones (As many as your region allows).
- 2 - 4 Public Subnets in different availablity zones (As many as your region allows).
- An Internet Gateway to route traffic to your public subnets.
- Route tables for all your subnets public and private

![diagram showing subnets](../assets/community/surrealdb-on-aws/diagram_1.png 'How I set up my subnets')

Things to keep in mind:

- Your public subnet route tables should have your internet gateway setup as a target and the destination should be 0.0.0.0/0.

![diagram showing public route table](../assets/community/surrealdb-on-aws/diagram_2.png 'How I set up my public route table')

- Your private subnets should only have access to your local network. (This is one example but all of them should look like this)

![diagram showing private route table](../assets/community/surrealdb-on-aws/diagram_3.png 'How I set up my private subnet route table')

</br>

# Settings up the TiKV Cluster

***This next part is only for those looking to setup a tikv storage layer cluster***

The simplest and most maintainable way to deploy a tikv cluster is to the the TiUP cli tool created by the TiKV foundation. This will require deploying the cluster on EC2 instances. Using this it will be very easy to scale in and out the cluster as is needed.

The first step is create a control EC2 instance, this instance is what you will ssh into to run the TiUP cli. This can be skipped if security is not a concern and you are only testing things out. However, best practice is to only have this one instance accessable from outside your vpc (i.e. on a public subnet) whilst all other instance are running on private subnets.

After clicking on launch instance, type a name for your instance:
![](../assets/community/surrealdb-on-aws/diagram_4.png '')

then select amazon linux as your distro (or any other linux distro also works):
![](../assets/community/surrealdb-on-aws/diagram_5.png '')

Select your instance type (for testing t2.micro is fine but you will probably want something more powerful for a production level deployment):
![](../assets/community/surrealdb-on-aws/diagram_6.png '')

Next create a key and download it. Keep this in a secure place because you will need this to ssh into all your instances:
![](../assets/community/surrealdb-on-aws/diagram_7.png '')

This is a very important step to pay close attention to the setup. Now we configure the network settings for this instance. The controller instance must be accessiable from the outside so be sure to put it in one of the public subnets. Also make sure to allow ssh connections from anywhere:
![](../assets/community/surrealdb-on-aws/diagram_8.png '')

For storage you can select the default 8GB as you will not need more than that.

With this you have finished creating your controller instance.The steps to create PD Nodes and Storage Nodes is identical except the security group and the network settings. Note here we are creating the instance with a new security group, this group will be for all placement drivers so when you create new placement drivers you can use this existing security group instead of creating a new one. Also note that the this node is being created on a private subnet which means it will only be accessable from other instances in the same vpc (hence why we created the controller instance). Lastly note that the ssh rule only allows connections on 10.0.0.0/16 basically this means that any instance with a private ip address of 10.0.*.* will be able to connect to this instance over port 22:
![](../assets/community/surrealdb-on-aws/diagram_9.png '')

Also for the data nodes remember to give them adequate storage (around 20GB for development and 40-100GB for production)

Finally create another instance which will be the monitoring instance hosting a grafana and promethus server to see the state of your cluster. Create a new security group for this too and make sure it is on a public subnet. This instance will require a fair amount of storage because it will keep all your logs so around 30-40GB is sufficient.

For testing and development 1 pd node is sufficient but for redundancy in production having 2 is advised.

For the data nodes follow the same steps and the pd nodes except create a seperate security group for them. Also for every node you create data or pd make sure to space them out evenly across all your private subnets and availablity zones.

Ok I know these are a lot of steps but we are nearly there.

The next thing we have to do is to configure the security groups to allow connections on the right ports.

For the placement driver security group configure these inbound rules for the security group:
![](../assets/community/surrealdb-on-aws/diagram_10.png '')

For the data nodes security group configure these inbound rules:
![](../assets/community/surrealdb-on-aws/diagram_11.png '')

For the monitor security group configure these inbound rules:
![](../assets/community/surrealdb-on-aws/diagram_12.png '')

at this point all the firewall and routing for your cluster is compelete (Actually we will create another security group for the SurrealDB nodes as well but that is later)

First thing we have to do is copy our ssh key to our control system so we can use it within the control instance to ssh into all the other instances.

Choose what ever method you like for this. The simplest way is to use:

`scp -i $PATH_TO_KEY $IP_ADDRESS_OF_CONTROL_INSTANCE $PATH_TO_KEY $PATH_TO_PUT_KEY_IN`

 command

 After doing that you can ssh into the control instance using the public ip address.
 Once inside verify you can ssh into all other instance from inside the control instance using the indentity file you copied over and the private ip addresses of all the nodes.

 From this point onwards the TiKV guide to use TiUP to create a cluster is very well documented so I won't bother repeating it here. Simply follow that guide and return once compelete

 <https://tikv.org/docs/6.5/deploy/install/production/>

After completing your TiKV cluster setup we have to setup load balancers for the placement drivers.

Again given the length of this tutorial I will spare the details of how to do this. Googling how to setup load balancers for EC2 instances will yield many articles and videos so pick your favorite and follow that.

<https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-application-load-balancer.html>

With this setup you have a fully function TiKV cluster on AWS

At this point you can either create another instance for the surrealdb node or deploy an AWS ECS cluster. For this Tutorial we will use an ECS cluster

# Deploying an AWS ECS cluster for SurrealDB

Start by creating an new Task Definition for the the surrealdb image
![](../assets/community/surrealdb-on-aws/diagram_13.png '')
![](../assets/community/surrealdb-on-aws/diagram_14.png '')

next create a new cluster in the same vpc as your tikv cluster. Make sure to only select public subnets or else your cluster won't be accessable be able to download the surrealdb image from docker hub.

You are almost finished, the last step is to create a load balancer and then a service that will run the surrealdb nodes (You can create the load balancer at the same time as you create your service). Again there are many tutorials out there to do this. You can also setup the service to auto scale if needed.

Thank you for reading this very long tutorial and hopefully it was helpful. If you have any questions or suggestions feel free to leave them on the surrealdb discord and they will be addressed promptly.
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

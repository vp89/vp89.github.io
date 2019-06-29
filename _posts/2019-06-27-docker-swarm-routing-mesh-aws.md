---
layout: post
title:  "Docker swarm mode routing mesh on AWS"
date:   2019-06-27 00:00:00 -0000
categories: programming
---

Docker has a feature named *swarm mode* which allows you to create a cluster of Docker engines with manager and worker nodes. The cluster has a single leader and commands issued to it are propagated to all the other nodes. I'm quite interested in this feature as it provides a lot of the more commonly used functionality of Kubernetes with less complexity and setup time.

One interesting *sub-feature* of swarm mode is the built-in routing mesh. This mesh allows you to define a `published port` on your cluster which maps to a `target port`. All nodes in the cluster can accept traffic on the `published port` and automatically route it to nodes which have active containers running and listening on the `target port`.

So for example, let's say you have a cluster of 5 nodes with the following IPs:

- 192.168.1.2
- 192.168.1.3
- 192.168.1.4
- 192.168.1.5
- 192.168.1.6

You also have a `Docker service` configured with a containerized application listening on port 80, set to use 3 replicas. With the routing mesh, you could hit either of the 2 hosts which isn't running that container and the traffic will be routed and load-balanced to one of the active nodes which is.

This is not supposed to act in place of a real load-balancer, but the two can work well together. For example the load balancer can be configured to include all the nodes in the cluster and then as you increase or decrease the number of replicas, you won't need to modify the load balancer config.

One requirement for this routing mesh to work is for the following ports to be enabled for ingress on all nodes in the cluster:

2377 TCP for swarm mode (this will be needed even if you don't use the routing mesh)

7946 TCP/UDP for container network discovery

4789 UDP for container ingress network

If you are trying this feature out in AWS with a DIY cluster of EC2 instances, you will likely need to create a security group with those rules and apply it to all the instances. Once that's done, the swarm mode with the routing mesh should work after a few seconds as those calls it makes on those ports will start going through.

Defining a port to be a `published port` on an existing service can be done in the following way:

```
docker service update --publish-add published=<PUBLISHED-PORT>,target=<CONTAINER-PORT> <SERVICE>
```

Check out the excellent [Docker documentation](https://docs.docker.com/engine/swarm/ingress/) for more details.

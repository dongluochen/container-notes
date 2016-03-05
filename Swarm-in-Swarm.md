#Swarm in Swarm

Use Docker-in-Docker(dind) and multi-host overlay network to create Swarm clusters with hundreds of dind nodes.

Swarm is designed to manage clusters with hundreds of nodes. It's expensive to run such tests with hundreds of nodes from cloud provider. This project is to create hundreds of dind nodes from a few EC2 instances. It first creates a Swarm cluster (swarm1) with a few EC2 instances; adds a CIDR /16 overlay network (ring2net); then creates a swarm cluster (swarm2) on ring2net; next adds docker instances to swarm2 through swarm1. Swarm2 can reach hundreds of nodes with overlay network capability. We want to create scripts to facilitate this procedure.

![Swarm in Swarm architecture](https://github.com/dongluochen/container-notes/blob/master/swarm-in-swarm.png)

This project is inspired by Docker Swarm integration test where dind is used to create swarm cluster for test. It's useful to stress swarm manager or K/V store, and examine system failure.

#Start swarm1

Get nodes from cloud provider. Here we use AWS EC2 as example. Images and volumes in Dind are not shared with host. Each Dind instance would may take some disk space from host. Nodes need larger storage space like 40GB for this experiment. Create a VPC and open network ports for external access and overlay network. In this example the hostnames of the EC2 instances are `dong-wily-k` where we use Ubuntu Wily images.

Select a node to start a discovery. Here we use consul as example. 
```bash
docker dong-wily-1:2377 run -d \
    -p "8500:8500" \
    -h "consul" \
    progrium/consul -server -bootstrap
```

Start nodes. For example, a node with IP 172.19.108.80 listens on port 2377 for Docker daemon. It includes cluster-store option to support overlay network. 
```bash
/usr/bin/docker daemon -H 172.19.108.80:2377 -H unix:///var/run/docker.sock --cluster-store=consul://172.19.29.201:8500/swarm --cluster-advertise=172.19.108.80:2377
```

Start swarm1 manager and join the nodes. 
```
ubuntu@ip-172-19-9-45:~$ docker -H dong-wily-1:2378 info
Containers: 509
 Running: 500
 Paused: 0
 Stopped: 9
Images: 13
Server Version: swarm/1.1.2
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 3
 ip-172-19-9-45: 172.19.9.45:2377
  └ Status: Healthy
  └ Containers: 364
  └ Reserved CPUs: 0 / 4
  └ Reserved Memory: 0 B / 15.42 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.2.0-22-generic, operatingsystem=Ubuntu 15.10, storagedriver=devicemapper
  └ Error: (none)
  └ UpdatedAt: 2016-03-04T21:56:13Z
 ip-172-19-29-201: 172.19.29.201:2377
  └ Status: Healthy
  └ Containers: 11
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 3.843 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.2.0-22-generic, operatingsystem=Ubuntu 15.10, storagedriver=aufs
  └ Error: (none)
  └ UpdatedAt: 2016-03-04T21:55:42Z
 ip-172-19-108-80: 172.19.108.80:2377
  └ Status: Healthy
  └ Containers: 134
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 3.843 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.2.0-22-generic, operatingsystem=Ubuntu 15.10, storagedriver=devicemapper
  └ Error: (none)
  └ UpdatedAt: 2016-03-04T21:56:04Z
Plugins: 
 Volume: 
 Network: 
Kernel Version: 4.2.0-22-generic
Operating System: linux
Architecture: amd64
CPUs: 6
Total Memory: 23.11 GiB
Name: 605bd1b4fbde
```

#Start Swarm 2

Note that swarm2 is created thru swarm1. K/V store, swarm manager, dind instances are created on a same overlay network `ring2net`. Here the swarm1 interface is `dong-wily-1:2378`.

Create overlay network `ring2net`.

```
docker -H dong-wily-1:2378 network create -d overlay --subnet=10.9.0.0/16 ring2net
ubuntu@ip-172-19-29-201:/$ docker -H dong-wily-1:2378 network inspect ring2net | grep -i 10.9
                    "Subnet": "10.9.0.0/16"
```

Start consul on ring2net network

```
ubuntu@ip-172-19-29-201:/$ docker -H dong-wily-1:2378 run -d --name=consul-ring2 --net=ring2net progrium/consul -server -bootstrap
0aea5a87f78ab52942e33e827e07ad200a56e505cc06a70de7a92ee0628d7e1f
```

Start a swarm manager on ring2net. The manager is exposed on port 2398 of its host. You can add node constraint to pin the manager to a specific machine.

```
ubuntu@ip-172-19-29-201:/$ docker -H dong-wily-1:2378 run -d -p 2398:2398 --net=ring2net --name=manager-ring2 swarm -l debug manage -H :2398 consul://10.9.0.2:8500/swarm 
da5b45a6529c37172ce0e7596dc3ae5aa6aeaf264bfec762c2aeea1e06a0417c
```

Start dind thru swarm1
```
ubuntu@ip-172-19-29-201:/$ docker -H dong-wily-1:2378 run --net=ring2net --name=node1-ring2 -h "node1-ring2" --privileged -e constraint:node==ip-172-19-9-45 -d docker:dind --cluster-store=consul://10.9.0.2:8500/swarm --cluster-advertise=eth0:2375
04c960ccec250069e70009964188af71e24b47886b2c346ae86a04bc38324f0a
```

Add nodes to swarm at ring2net
```
ubuntu@ip-172-19-108-80:~$ docker -H dong-wily-1:2378 run -d --net=ring2net --name=join3-ring2net swarm join --addr=node1-ring2:2375 consul://10.0.0.2:8500/swarm
ee3d988dd28a5d3bd79dc7cbf729e4015a76270134dead2afbb7ddb5d0031e60
```

Use script to repeat the previous 2 steps to add nodes to swarm2.
```
ubuntu@ip-172-19-108-80:~$ docker -H dong-wily-1:2398 info 
Containers: 4
 Running: 1
 Paused: 0
 Stopped: 3
Images: 3
Server Version: swarm/1.1.2
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 148
 (unknown): node93-ring2:2375
  └ Status: Pending
  └ Containers: 0
  └ Reserved CPUs: 0 / 0
  └ Reserved Memory: 0 B / 0 B
  └ Labels: 
  └ Error: (none)
  └ UpdatedAt: 2016-03-03T22:44:46Z
 node1-ring2: node1-ring2:2375
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 4
  └ Reserved Memory: 0 B / 15.42 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.2.0-22-generic, operatingsystem=Alpine Linux v3.3 (containerized), storagedriver=vfs
  └ Error: (none)
  └ UpdatedAt: 2016-03-03T22:44:23Z
…
```

Now you have swarm2 with large nubmer of nodes.

#Notes

Consul becomes a bottleneck in a large cluster where it may not be able to handle the load. You can put consul in a node with plenty of resource.

Monitor disk space at your nodes.

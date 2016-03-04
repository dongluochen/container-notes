#Swarm in Swarm

Use Docker-in-Docker(dind) and multi-host overlay network to create Swarm clusters with hundreds of dind nodes.

Swarm is designed to manage clusters with hundreds of nodes. It's expensive to run such tests with hundreds of nodes from cloud provider. This project is to create hundreds of dind nodes from a few EC2 instances. It first creates a Swarm cluster (Ring1) with a few EC2 instances; adds a CIDR /16 overlay network (ring2net); then creates a swarm cluster (Ring2) on ring2net; next adds docker instances to Ring2 through Ring1. Ring2 can reach hundreds of nodes with overlay network capability. We want to create scripts to facilitate this procedure.

![Swarm in Swarm architecture](https://github.com/dongluochen/container-notes/blob/master/warm-in-swarm.pdf)

This project is inspired by Docker Swarm integration test where dind is used to create swarm cluster for test. 

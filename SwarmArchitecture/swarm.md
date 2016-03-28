#Swarm Architecture

A Swarm cluster has 3 components, multiple swarm managers to provide high availability, a discovery backend for node discovery and leader election for managers, and a group of Docker nodes to run user containers.

![swarm diag](https://github.com/dongluochen/container-notes/blob/master/SwarmArchitecture/swarmArchitecture.png)

Swarm works closely with Docker tools. You can use Docker Machine to create the Swarm cluster, use Docker Compose to define a complex user scenario, and use public or private Docker Registry to download Docker images.

![swarm tools](https://github.com/dongluochen/container-notes/blob/master/SwarmArchitecture/swarm-tooling.png)

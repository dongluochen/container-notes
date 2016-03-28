#How to Setup Swarm in Azure

This doc describes the steps to set up a Swarm cluster in Microsoft Azure cloud. It uses Azure template to create a Swarm cluster, adds Windows Swarm manager to the cluster, and adds Windows Server 2016 Technical Preview with Container support as worker nodes to Swarm. 

##Start a Swarm cluster in Azure

Microsoft has made Azure template for [Swarm cluster](https://github.com/Azure/azure-quickstart-templates/tree/master/docker-swarm-cluster). It uses `Consul` as discovery backend, supports high availability with multiple manager, and deploys CoreOS as host operating system for managers and nodes. The following link is from the github template.

![Azure template](https://github.com/Azure/azure-quickstart-templates/blob/master/docker-swarm-cluster/img/cluster-network.png)

There are 2 ways to deploy a Swarm cluster using the template. The template provides a `Deploy to Azure` link to start the work on Azure Portal. Ahmet Alp BAlkan has created a [video clip](https://channel9.msdn.com/Blogs/Regular-IT-Guy/Docker-Swarm-Clusters-on-Azure) on Channel 9 to demonstrate how to use the template. Alternatively users can do the same task with [Azure CLI](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-command-line-tools/). The commands are basically direct mapping from Portal operations. 

When you finish the deployment, use the SSH tunnel to connect to your swarm cluster. You can SSH into your Swarm managers or nodes to inspect or upgrade them. 

```
Dongluos-MacBook-Pro:.ssh dongluochen$ docker info
Containers: 6
Images: 10
Role: replica
Primary: 10.0.0.5:2375
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 4
 swarm-node-0: 192.168.0.5:2375
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.2.0 (Coeur Rouge), storagedriver=overlay
 swarm-node-1: 192.168.0.6:2375
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.2.0 (Coeur Rouge), storagedriver=overlay
 swarm-node-2: 192.168.0.4:2375
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.2.0 (Coeur Rouge), storagedriver=overlay
 swarm-node-3: 192.168.0.7:2375
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.1.0 (Coeur Rouge), storagedriver=overlay
CPUs: 8
Total Memory: 14.13 GiB
Name: 2176d1c1eeb0
```

## Add a Windows Swarm Manager

### Build Swarm Binary for Windows

To build Swarm binary for Windows, get Swarm source code from Github, build with `GOOS=windows` option. Swarm code supports Linux, Windows and Mac platform. 

```
$git clone git@github.com:docker/swarm.git

$cd swarm

$GOOD=windows godep go build

$ ls -l swarm.exe
-rwxr-xr-x  1 dongluochen  staff  25825280 Mar 28 11:46 swarm.exe

```

### Add Windows Swarm manager

Add a Windows Server 2012R2 to Swarm manager network. Login the server through RDP session (See [instructions](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-windows-classic-connect-logon/) from Azure). Open firewall ports for Swarm communication.  

```
netsh advfirewall firewall add rule name="docker swarm" dir=in action=allow protocol=tcp localport=2375 enable=yes

```

You can run Windows Swarm manager in standalone mode. Here we add it to the existing HA manager list. Copy swarm.exe binary to the Windows server (10.0.0.8). The command to start a Windows manager is the same as other servers. 

```
PS C:\Users\dchen> .\swarm.exe -l debug manage -H 0.0.0.0:2375 --replication --advertise 10.0.0.8:2375 -discovery-opt kv.path=docker/nodes consul://10.0.0.4:8500
...
time="2016-03-28T19:52:50Z" level=info msg="Listening for HTTP" addr="0.0.0.0:2375" proto=tcp
time="2016-03-28T19:52:50Z" level=debug msg="Watch triggered with 5 nodes" discovery=consul
time="2016-03-28T19:52:50Z" level=info msg="Leader Election: Cluster leadership lost"
time="2016-03-28T19:52:50Z" level=info msg="New leader elected: 10.0.0.5:2375"
time="2016-03-28T19:52:50Z" level=debug msg="Watch triggered with 5 nodes" discovery=consul
time="2016-03-28T19:52:50Z" level=debug msg="Start monitoring events" id="OGHH:UEJW:OQBG:ONYD:H43W:DZ6E:SQ7T:2HZA:2DSA:6BSR:VCLJ:4JQF" name=swarm-node-0
time="2016-03-28T19:52:50Z" level=debug msg="Start monitoring events" id="KKTD:SWE7:FO24:JX2Z:3Z2P:KTSO:MIPA:F2DQ:5E4R:3XWZ:BDFF:WSIJ" name=swarm-node-1
time="2016-03-28T19:52:50Z" level=debug msg="Start monitoring events" id="PIYJ:OJ3Q:KHRN:DZUC:5YLF:5WVL:NSMR:7R4X:K5LW:EDPS:GBFU:XXOJ" name=swarm-node-3
time="2016-03-28T19:52:50Z" level=debug msg="Start monitoring events" id="22LT:NHK7:IYFW:Z634:2VNM:HKP2:6CLU:XAGO:Y55U:QZIZ:4WS3:B66R" name=swarm-node-2
time="2016-03-28T19:52:50Z" level=info msg="Registered Engine swarm-node-0 at 192.168.0.5:2375"
time="2016-03-28T19:52:50Z" level=info msg="Registered Engine swarm-node-2 at 192.168.0.4:2375"
time="2016-03-28T19:52:50Z" level=info msg="Registered Engine swarm-node-3 at 192.168.0.7:2375"
time="2016-03-28T19:52:50Z" level=info msg="Registered Engine swarm-node-1 at 192.168.0.6:2375"
...
```

Docker `info` to this new manager. Notice it identifies `10.0.0.5:2375` as Primary and its kernel version is `6.2 9200 (9600.18146.amd64fre.winblue_ltsb.151121-0600)`, a Windows 2012R2 kernel. 

```
azureuser@swarm-master-1 ~ $ docker -H 10.0.0.8:2375 info
Containers: 6
Images: 10
Server Version: swarm/1.2.0
Role: replica
Primary: 10.0.0.5:2375
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 4
 swarm-node-0: 192.168.0.5:2375
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.2.0 (Coeur Rouge), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-03-28T19:55:13Z
 swarm-node-1: 192.168.0.6:2375
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.2.0 (Coeur Rouge), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-03-28T19:55:14Z
 swarm-node-2: 192.168.0.4:2375
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.2.0 (Coeur Rouge), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-03-28T19:55:13Z
 swarm-node-3: 192.168.0.7:2375
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.2.0 (Coeur Rouge), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-03-28T19:55:14Z
Kernel Version: 6.2 9200 (9600.18146.amd64fre.winblue_ltsb.151121-0600)
Operating System: windows
CPUs: 8
Total Memory: 14.13 GiB
Name: swarm-manager

azureuser@swarm-master-1 ~ $ docker -H 10.0.0.8:2375 run hello-world

Hello from Docker.
This message shows that your installation appears to be working correctly.
...
```

## Add Windows node to Swarm

Windows Server 2016 Technical Preview 4 supports Docker daemon. In your Azure resource group, add a new Windows Server by selecting 'Windows Server 2016 Core with Container Tech Perview 4' and put it in `subnet-nodes`. 

When the node is provisioned, you can connect to it through RDP session. Open firewall ports for Docker daemon and overlay network. 

```
Netsh advfirewall firewall add rule name="docker daemon 1" dir=in action=allow protocol=tcp localport=2375,7946 enable=yes

Netsh advfirewall firewall add rule name="docker daemon 2" dir=in action=allow protocol=udp localport=4789,7946 enable=yes

```

Start Docker daemon on the server. `docker.exe` is already added at `C:\Windows\System32`.


```
C:\windows\system32>docker.exe daemon -H 0.0.0.0:2375 --cluster-store=10.0.0.4:8500 --cluster-advertise=192.168.0.8:2375 

```

The new Windows node shows up in Swarm cluster. 

```
azureuser@swarm-master-0 ~ $ docker -H swarm-master-1:2375 info
Containers: 2
Images: 6
Role: replica
Primary: 10.0.0.5:2375
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 5
 docker-tp4-2: 192.168.0.8:2375
  └ Status: Healthy
  └ Containers: 0
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 7.35 GiB
  └ Labels: executiondriver=
 Name: Windows 1854
 Build: 1.10.0-dev 59a341e
 Default Isolation: process, kernelversion=10.0 10586 (10586.0.amd64fre.th2_release.151029-1700), operatingsystem=Windows Server 2016 Technical Preview 4, storagedriver=windowsfilter
 swarm-node-0: 192.168.0.5:2375
  └ Status: Healthy
  └ Containers: 0
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.1.0 (Coeur Rouge), storagedriver=overlay
 swarm-node-1: 192.168.0.6:2375
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.1.0 (Coeur Rouge), storagedriver=overlay
 swarm-node-2: 192.168.0.4:2375
  └ Status: Healthy
  └ Containers: 1
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.1.0 (Coeur Rouge), storagedriver=overlay
 swarm-node-3: 192.168.0.7:2375
  └ Status: Healthy
  └ Containers: 0
  └ Reserved CPUs: 0 / 2
  └ Reserved Memory: 0 B / 3.532 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.4.6-coreos, operatingsystem=CoreOS 991.1.0 (Coeur Rouge), storagedriver=overlay
CPUs: 10
Total Memory: 21.48 GiB
Name: dfe6ec556ea6
```



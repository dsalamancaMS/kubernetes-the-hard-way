# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [region](https://azure.microsoft.com/en-us/global-infrastructure/regions/).

> Ensure a default resource group and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Network

In this section a dedicated [Virtual Network](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview) VNET network will be setup to host the Kubernetes cluster.

Create the `kubernetes-hard-vnet` Virtual Network:

```
az network vnet create \
	--name kube-hard-vnet \
	--address-prefix 10.0.0.0/8
```

A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kube-nodes-sub` subnet in the `kube-hard-vnet` network:

```
az network vnet subnet create \
	--vnet-name kubernetes-hard-vnet \
	-n kube-nodes-sub \
	--address-prefix 10.240.0.0/24
```

> The `10.240.0.0/24` IP address range can host up to 251 virtual machines.

### Firewall Rules/ Network Security Groups

[Network Security Groups](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview) in Azure serve as a virtual firewalls for controlling inbound and outbound traffic.

Create an NSG for our recently created VNET:

```
az network nsg create \
	-n kube-hard-nsg
```

Let's attach now our NSG to the `kube-nodes-sub` subnet:

```
az network vnet subnet update \
	--name kube-nodes-sub \
	--vnet-name kubernetes-hard-vnet \
	--network-security-group kube-hard-nsg
```
Update NSG rules to allow external SSH, and HTTPS:

```
az network nsg rule create \
	--nsg-name Kube-Hard-NSG \
	-n allow_ssh_https \
	--priority 100 \
	--source-address-prefixes '*' \
	--source-port-ranges '*' \
	--destination-address-prefixes '*' \
	--destination-port-ranges 22 6443 \
	--access Allow \
	--protocol Tcp \
	--description "Allow SSH and HTTPS from any Source."
```
>You can modify the `--source-address-prefixes` with your own IP address to increase security of your cluster.

List the NSG configuration, making sure that the subnet and the newly created rules show up:

```
az network nsg show -n kube-hard-nsg
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
az network public-ip create \
	-n kube-hard-public-api-ip \
	--allocation-method Static
```

Verify the `kube-hard-public-api-ip` static IP address was created in your default region:

```
az network public-ip show \
-n kube-hard-public-api-ip \
--query "{name: name, resourceGroup: resourceGroup, address: ipAddress, AllocationMethod: publicIpAllocationMethod}" \
-o table
```

> output

```
Name                     ResourceGroup    Address        AllocationMethod
-----------------------  ---------------  -------------  ------------------
kube-hard-public-api-ip  KubeHardWay      xx.xx.xx.xx  Static
```

## Virtual Machines

The virtual machines in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each virtual machine will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

>Before creating the virtual machines we will be creating an Availability set that will contain our K8s controllers in order to provide redundancy and high availability.

## Availability Set

Availability sets are a group of virtual machines that are deployed across fault domains and update domains. Availability sets will make sure that our K8s Controllers will not be affected by single points of failure, like the network switch or the power unit of a rack of servers.

```
az vm availability-set create \
	-n kube-controllers
```

### Creating the Kubernetes Controllers

Create three virtual machines which will host the Kubernetes control plane:

>We will be using B1s sized VMs to try and stay on free tier as possible.
>Password must have the following: 1 lower case character, 1 upper case character, 1 number and 1 special character
```
for i in 0 1; do
  az vm create \
	-n kube-controller-${i} \
	--image Canonical:UbuntuServer:18.04-LTS:18.04.201805220 \
	--size Standard_B1s \
	--availability-set kube-controllers \
	--vnet-name kubernetes-hard-vnet \
	--subnet kube-nodes-sub \
	--private-ip-address 10.240.0.1${i} \
	--nsg kube-hard-nsg \
	--storage-sku Standard_LRS \
	--authentication-type password \
	--admin-username example_user \
	--admin-password Example_pass123!
done
```

### Kubernetes Workers

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
 az vm create \
	-n kube-node-${i} \
	--image Canonical:UbuntuServer:18.04-LTS:18.04.201805220 \
	--size Standard_B1s \
	--vnet-name kubernetes-hard-vnet \
	--subnet kube-nodes-sub \
	--private-ip-address 10.240.0.2${i} \
	--nsg kube-hard-nsg \
	--storage-sku Standard_LRS \
	--authentication-type password \
	--admin-username example_user \
	--admin-password Example_pass123!
done
```
Enable IP forwarding on node NICs

```
for i in 0 1 2; do 
az network nic update \
-n $(az vm show --name kube-node-${i} --query [networkProfile.networkInterfaces[*].id] --output tsv | sed 's:.*/::') \
--ip-forwarding true
done
```
### Verification

List the virtual machines in your default resource group:

```
az vm list -d \
--query '[].{name: name, ipAddress: privateIps, PublicIP: publicIps, Version: storageProfile.imageReference.sku, status: powerState}' \
-o table
```

> output

```
Name                IpAddress    PublicIP        Version    Status
------------------  -----------  --------------  ---------  ----------
kube-controller-00  10.240.0.10  xx.xxx.xxx.xxx  18.04-LTS  VM running
kube-controller-01  10.240.0.11  xx.xxx.xxx.xxx  18.04-LTS  VM running
kube-node-0         10.240.0.20  xx.xxx.xxx.xxx  18.04-LTS  VM running
kube-node-1         10.240.0.21  xx.xxx.xxx.xxx  18.04-LTS  VM running
kube-node-2         10.240.0.22  xx.xxx.xxx.xxx  18.04-LTS  VM running
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)

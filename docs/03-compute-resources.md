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

>Before creating the virtual machines we will be creating an Availability set that will contain our K8s controllers in order to provide High availability.

## Availability Set

Availability sets are a group of virtual machines that are deployed across fault domains and update domains. Availability sets will make sure that our K8s Controllers will not be affected by single points of failure, like the network switch or the power unit of a rack of servers.

###### Continue Here

### Creating the Kubernetes Controllers

Create three virtual machines which will host the Kubernetes control plane:

```
for i in 0 1; do
  
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### Verification

List the compute instances in your default compute zone:

```
gcloud compute instances list
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as describe in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `controller-0` compute instances:

```
gcloud compute ssh controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `controller-0` instance:

```
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-1006-gcp x86_64)

...

Last login: Sun May 13 14:34:27 2018 from XX.XXX.XXX.XX
```

Type `exit` at the prompt to exit the `controller-0` compute instance:

```
$USER@controller-0:~$ exit
```
> output

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)

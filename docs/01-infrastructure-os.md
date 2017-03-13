# Cloud Infrastructure Provisioning - OpenStack

This lab will walk you through provisioning the compute instances required for running a H/A Kubernetes cluster. A total of 6 virtual machines will be created.

After completing this guide you should have the following compute instances:

```
openstack server list
```

````
FILL ME IN
````

> All machines will be provisioned with fixed private IP addresses to simplify the bootstrap process.

To make our Kubernetes control plane remotely accessible, a public IP address will be provisioned and assigned to a Load Balancer that will sit in front of the 3 Kubernetes controllers.

## Networking

Create a Kubernetes network:

```shell
neutron net-create KUBERNETES_NET
```

Create a subnet for the Kubernetes cluster:

```shell
neutron subnet-create KUBERNETES_NET 10.240.0.0/24 --name kubernetes
```

### Firewall Rules

Create a security group for our kubernetes firewall rules

```shell
neutron security-group-create kubernetes --description "Kubernetes firewall rules"
```

Allow all instances to be ping

```shell
neutron security-group-rule-create --protocol icmp kubernetes
```

```
neutron security-group-rule-create --direction ingress --protocol tcp --port-range-min 3389 --port-range-max 3389 kubernetes
```

```
neutron security-group-rule-create --direction ingress --protocol tcp --port-range-min 22 --port-range-max 22 kubernetes
```

```
neutron security-group-rule-create --direction ingress --protocol tcp --port-range-min 6443 --port-range-max 6443 kubernetes
```

```
neutron security-group-list
+--------------------------------------+------------+----------------------------------------------------------------------+
| id                                   | name       | security_group_rules                                                 |
+--------------------------------------+------------+----------------------------------------------------------------------+
| 7aa8341d-bc8c-4fbd-a1ca-7f8e0798b3bb | kubernetes | egress, IPv4                                                         |
|                                      |            | egress, IPv6                                                         |
|                                      |            | ingress, IPv4, 22/tcp                                                |
|                                      |            | ingress, IPv4, 3389/tcp                                              |
|                                      |            | ingress, IPv4, 6443/tcp                                              |
|                                      |            | ingress, IPv4, icmp                                                  |
+--------------------------------------+------------+----------------------------------------------------------------------+
```

If you can not reach external networks you might need to create a router and attach it to the kubernetes network.

```shell
(neutron) router-create  KUBERNETES_NET_ROUTER
```

```shell
neutron router-gateway-set KUBERNETES_NET_ROUTER GATEWAY_NET
```

```shell
neutron router-interface-add KUBERNETES_NET_ROUTER kubernetes
```

```shell
Set gateway for router KUBERNETES_NET_ROUTER
```

### Kubernetes Public Address

Create a public IP address that will be used by remote clients to connect to the Kubernetes control plane:

```shell
neutron floatingip-create <Replace with your Network that has public IP's>
```

```shell
neutron floatingip-list
```

```shell
+--------------------------------------+------------------+---------------------+---------+
| id                                   | fixed_ip_address | floating_ip_address | port_id |
+--------------------------------------+------------------+---------------------+---------+
| d6f3ed33-a66b-49bc-826b-f24ba6c42091 |                  | 10.242.0.159        |         |
+--------------------------------------+------------------+---------------------+---------+
```

## Provision Virtual Machines

All the VMs in this lab will be povissioned using CoreOS (container ) because we can download the CoreOS image ensuring we all have a consistant starting point.

### Install CoreOS Container Linux image

We will be using the beta chanel image, you can go [here](https://coreos.com/os/docs/latest/booting-on-openstack.html) to try out other images.

```shell
wget https://beta.release.core-os.net/amd64-usr/current/coreos_production_openstack_image.img.bz2
bunzip2 coreos_production_openstack_image.img.bz2

glance image-create --name CoreOS-beta \
--container-format bare \
--disk-format qcow2 \
--file coreos_production_openstack_image.img

+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 331c8bf5b3e89ef46be139968a447153     |
| container_format | bare                                 |
| created_at       | 2017-03-07T19:59:45Z                 |
| disk_format      | qcow2                                |
| id               | a79be365-a49d-4d01-a5dd-f6e5bfae3e82 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | CoreOS-beta                          |
| owner            | db16b4670f274486814b29fc89761bc5     |
| protected        | False                                |
| size             | 801177600                            |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2017-03-07T20:00:04Z                 |
| virtual_size     | None                                 |
| visibility       | private                              |
+------------------+--------------------------------------+
```

### Virtual Machines


Requirements:
  * image id
  * flavor id
  * keypair name
  * network internal id: d5519e79-e0d7-4480-a703-5af94e9328ae
  * network kubernetes id: 9e6b8924-5fd4-4afd-af5c-6f219f255248

#### Kubernetes Controllers

```shell
nova boot --image CoreOS-beta --flavor k8c.small  \
--nic net-name=INTERNAL_NET \
--nic net-name=KUBERNETES_NET,v4-fixed-ip=10.240.0.10 \
--key-name k8s --security-groups default,kubernetes controller0
```

```shell
nova boot --image CoreOS-beta --flavor k8c.small  \
--nic net-name=INTERNAL_NET \
--nic net-name=KUBERNETES_NET,v4-fixed-ip=10.240.0.11 \
--key-name k8s --security-groups default,kubernetes controller1
```

```shell
nova boot --image CoreOS-beta --flavor k8c.small  \
--nic net-name=INTERNAL_NET \
--nic net-name=KUBERNETES_NET,v4-fixed-ip=10.240.0.12 \
--key-name k8s --security-groups default,kubernetes controller2
```

#### Kubernetes Workers

```shell
nova boot --image CoreOS-beta --flavor k8m.small  \
--nic net-name=INTERNAL_NET \
--nic net-name=KUBERNETES_NET,v4-fixed-ip=10.240.0.20 \
--key-name k8s --security-groups default,kubernetes worker0
```

```shell
nova boot --image CoreOS-beta --flavor k8m.small  \
--nic net-name=INTERNAL_NET \
--nic net-name=KUBERNETES_NET,v4-fixed-ip=10.240.0.21 \
--key-name k8s --security-groups default,kubernetes worker1
```

```shell
nova boot --image CoreOS-beta --flavor k8m.small  \
--nic net-name=INTERNAL_NET \
--nic net-name=KUBERNETES_NET,v4-fixed-ip=10.240.0.22 \
--key-name k8s --security-groups default,kubernetes worker2
```
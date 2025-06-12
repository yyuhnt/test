# OpenStack Multi-Homed Routing Topology: Step-by-Step Guide

  

This guide walks you through building the multi-network, multi-homed VM topology as shown in your diagram, including both OpenStack Neutron configuration and required guest OS (Linux VM) settings for routing and NAT.

  

---

  

## A. OpenStack (Neutron) Setup

  

### 1. Create Networks and Subnets

  

```bash

# External/public network

openstack network create --external --provider-physical-network physnet1 --provider-network-type flat external-net

openstack subnet create --network external-net --allocation-pool start=172.24.4.150,end=172.24.4.200 --no-dhcp --gateway 172.24.4.1 --subnet-range 172.24.4.0/24 external-subnet

  

# DMZ

openstack network create dmz2-net

openstack subnet create --network dmz2-net --subnet-range 10.0.0.0/24 --gateway 10.0.0.1 dmz2-subnet

  

# API Gateway Net

openstack network create api-gateway-net

openstack subnet create --network api-gateway-net --subnet-range 10.0.10.0/24 --gateway 10.0.10.1 api-gateway-subnet

  

# Private1

openstack network create private-net1

openstack subnet create --network private-net1 --subnet-range 10.0.20.0/24 --gateway 10.0.20.1 private1-subnet

  

# Private2

openstack network create private-net2

openstack subnet create --network private-net2 --subnet-range 10.0.30.0/24 --gateway 10.0.30.1 private2-subnet

```

  

---

  

### 2. Create and Configure the Neutron Router

  

```bash

openstack router create external-router

openstack router set external-router --external-gateway external-net

openstack router add subnet external-router dmz2-subnet

```

  

> Only attach dmz2-net to the router per your topology.

  

---

  

### 3. (Optional) Create Keypairs, Flavors, and Security Groups

  

```bash

openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey

openstack flavor create --ram 2048 --vcpus 2 --disk 20 myflavor

openstack security group create allow-all

openstack security group rule create --proto icmp --ethertype IPv4 allow-all

openstack security group rule create --proto tcp --dst-port 22 allow-all

openstack security group rule create --proto tcp --dst-port 80 allow-all

openstack security group rule create --proto tcp --dst-port 443 allow-all

openstack security group rule create --proto tcp --dst-port 1:65535 allow-all

```

  

---

  

### 4. Boot the VMs and Attach NICs

  

```bash

# ai-instance (DMZ)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show dmz2-net -f value -c id) \

  --key-name mykey --security-group allow-all ai-instance

  

# endpoint-instance (DMZ + API Gateway)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show dmz2-net -f value -c id) \

  --nic net-id=$(openstack network show api-gateway-net -f value -c id) \

  --key-name mykey --security-group allow-all endpoint-instance

  

# api-gateway-instance (API Gateway + Private1 + Private2)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show api-gateway-net -f value -c id) \

  --nic net-id=$(openstack network show private-net1 -f value -c id) \

  --nic net-id=$(openstack network show private-net2 -f value -c id) \

  --key-name mykey --security-group allow-all api-gateway-instance

  

# private1-instance (Private1)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show private-net1 -f value -c id) \

  --key-name mykey --security-group allow-all private1-instance

  

# private2-instance (Private2)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show private-net2 -f value -c id) \

  --key-name mykey --security-group allow-all private2-instance

```

  

---

  

### 5. Assign Floating IPs for External Access (if needed)

  

```bash

openstack floating ip create external-net

openstack server add floating ip ai-instance <your-allocated-floating-ip>

```

  

---

  

## B. Guest OS Configuration for Inter-Subnet Routing

  

### 1. Enable IP Forwarding on Multi-Homed VMs

  

```bash

sudo sysctl -w net.ipv4.ip_forward=1

# Make permanent:

echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

sudo sysctl -p

```

  

### 2. Configure Routing on Private VMs

  

#### For VMs in private-net1 (10.0.20.0/24):

  

```bash

sudo ip route replace default via 10.0.20.1

```

  

#### For VMs in private-net2 (10.0.30.0/24):

  

```bash

sudo ip route replace default via 10.0.30.1

```

  

#### For VMs in api-gateway-net (if needed):

  

```bash

sudo ip route replace default via 10.0.10.1

```

  

---

  

### 3. Add NAT on api-gateway-instance (if VMs need Internet via DMZ)

  

```bash

# Identify interfaces

i p a

# Suppose: ens3 = api-gateway-net, ens4 = private-net1, ens5 = private-net2

# Enable NAT for outgoing traffic

sudo iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE

# Save iptables rules

sudo iptables-save | sudo tee /etc/iptables/rules.v4

```

  

---

  

### 4. Test the Setup

  

* From private1-instance and private2-instance, ping the gateway and external targets.

* Use traceroute to verify traffic path.

* Check iptables -t nat -L and sysctl net.ipv4.ip\_forward on all gateway VMs.

  

---

  

## C. Optional: Harden and Automate

  

* Use cloud-init or Ansible to automate guest OS config.

* Restrict allowed ports with security groups and/or OS firewalls.

* Document all static routes and gateways for clarity.

  

---

  

## Summary Table

  

| VM/Instance          | Networks Attached             | IP Forwarding | Gateway for          | Needs NAT        |

| -------------------- | ----------------------------- | ------------- | -------------------- | ---------------- |

| ai-instance          | dmz2-net                      | No            | N/A                  | No               |

| endpoint-instance    | dmz2-net, api-gateway-net     | Yes           | api-gateway-net      | Maybe            |

| api-gateway-instance | api-gateway-net, priv1, priv2 | Yes           | priv1, priv2         | Yes (for egress) |

| private1-instance    | priv1                         | No            | api-gateway-instance | N/A              |

| private2-instance    | priv2                         | No            | api-gateway-instance | N/A              |

  

---

  

You now have an OpenStack topology with multiple subnets, a Neutron router, and VMs configured for inter-subnet routing using Linux multi-homed gateway VMs.
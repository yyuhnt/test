### 1. Create Networks and Subnets

```sh
# External/public network

openstack network create --external --provider-physical-network physnet1 --provider-network-type flat external-net

openstack subnet create --network external-net --allocation-pool start=172.24.4.150,end=172.24.4.200 --no-dhcp --gateway 172.24.4.1 --subnet-range 172.24.4.0/24 external-subnet

# DMZ

openstack network create DMZ-network

openstack subnet create --network DMZ-network --subnet-range 172.24.4.32/27 --gateway 172.24.4.33 dmz-subnet

# private-control
openstack network create private-control

openstack subnet create --network private-control --subnet-range 10.100.15.0/24 --gateway 10.100.15.80 api-gateway-subnet

# waf-netowork
openstack network create waf-network

openstack subnet create --network waf-network --subnet-range 10.100.20.0/24 --gateway 10.100.20.80 waf-subnet


# region1

openstack network create application_region_1

openstack subnet create --network application_region_1 --subnet-range 10.100.30.0/24 --gateway 10.100.30.80 region1-subnet

  

# region2

openstack network create application_region_2

openstack subnet create --network application_region_2 --subnet-range 10.0.31.0/24 --gateway 10.0.31.80 private2-subnet

```


### 2. Create and Configure the Neutron Router
```bash

openstack router create external-router

openstack router set external-router --external-gateway external-net

openstack router add subnet external-router dmz-subnet
```


### 3. Create Keypairs, Flavor

```
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
openstack flavor create --ram 2048 --vcpus 1 --disk 20 m1.small.u22.04.fullhost 
openstack flavor create --ram 2048 --vcpus 2 --disk 20 m1-med-30-4G
openstack security group create allow-all
openstack security group rule create --proto tcp --dst-port 1:65535 allow-all
openstack security group rule create --proto icmp --ethertype IPv4 allow-all
```

### 4. Boot the VMs and Attach NICs

```bash
# api-gateway (DMZ + private-control + application_region_2)

openstack server create --image Ubuntu2204 --flavor m1.small.u22.04.fullhost \
  --nic net-id=$(openstack network show DMZ-network -f value -c id) \
  --nic net-id=$(openstack network show private-control -f value -c id) \
  --nic net-id=$(openstack network show application_region_2 -f value -c id) \
  --key-name mykey --security-group allow-all api-gateway

  

# NFW (private-control + waf-network)

openstack server create --image OPNSense --flavor myflavor \

  --nic net-id=$(openstack network show private-control -f value -c id) \

  --nic net-id=$(openstack network show waf-network -f value -c id) \

  --key-name mykey --security-group allow-all NFW

# ML_Controller (private-control + waf-network)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show private-control -f value -c id) \

  --key-name mykey --security-group allow-all ML_Controller

# WAF (waf-network + application_region_1)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show api-gateway-net -f value -c id) \

  --nic net-id=$(openstack network show application_region_1 -f value -c id) \

  --key-name mykey --security-group allow-all api-gateway-instance

  

# application1 (application_region_1)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show application_region_1 -f value -c id) \

  --key-name mykey --security-group allow-all application1

  

# application2 (application_region_2)

openstack server create --image Ubuntu2204 --flavor myflavor \

  --nic net-id=$(openstack network show application_region_2 -f value -c id) \

  --key-name mykey --security-group allow-all application2
```

### 5. Assign Floating IPs for External Access

```bash
openstack floating ip create external-net

openstack server add floating ip ai-instance <your-allocated-floating-ip>
```

## B. Guest OS Configuration for Inter-Subnet Routing
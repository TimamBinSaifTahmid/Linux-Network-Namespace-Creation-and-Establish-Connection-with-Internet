# Creating Linux Bridge Network between Two Namespace and Establish Egress Connection
This is a step by step documentation that describe the way to create two network namespace namely green and red and connect them
using a linux bridge network br0 and establish a egress connection from network namespaces (green, red) to any google public ip address (8.8.8.8).
<br><br>
## Step 1: Installing Prerequisite Linux Packages:

```
sudo apt update
sudo apt install iproute2 net-tools tcpdump iputils-ping iptables
```
here first we need to fetch up to date packages of linux using apt update
then we can install our required packages
<br><br>
## Step 2: Configuring a Linux Bridge:
**First we need to create a linux bridge named br0**

```
sudo ip link add dev br0 type bridge
```

<br>

**We can check if the bridge created successfully or not by the following command**

```
sudo ip link list
```

using the above command we can see that br0 interface is shown in interface list

<br>

**We need to change state of our newly created bridge br0 to up**
```
sudo ip link set dev br0 up
```
<br>

**In order to communicate through bridge interface we need to assign ip address to bridge br0**
```
sudo ip addr add 10.0.0.1/24 dev br0
```
<br>

**We can check if the ip address assigned correctly in bridge br0 by following command**
```
sudo ip addr
```
<br><br>


## Step 3: Configuring Linux Network Namespaces:
**First we need to create a two linux namespace namely green and red**
```
sudo ip netns add green
sudo ip netns add red
```
<br>

**we can see the namespace list to make sure the green and red namespace actually created**
```
sudo ip netns list
```
<br><br>

## Step 4: Creating Veth Cable to Connect Namespaces with the Bridge:
**Need to create two veth cable**
```
sudo ip link add veth-green-ns type veth peer name veth-green-br
sudo ip link add veth-red-ns type veth peer name veth-red-br
```
<br>

**We can now see 4 extra interface in interfaces list as each cable contains two different interface**
```
sudo ip link list
```
<br><br>

## Step 5: Moving One End of Each Veth Cable to Each Network Namespace:
**We need to move veth-green-ns and veth-red-ns side of veth cable to green and red namespace respectively**
```
sudo ip link set veth-green-ns netns green
sudo ip link set veth-red-ns netns red
```
<br><br>

## Step 6: Connect Other End of Each Cable with Bridge:
**We need to connect other end of each cable veth-green-br and veth-red-br with bridge br0**
```
sudo ip link set dev veth-green-br master br0
sudo ip link set dev veth-red-br master br0
```
<br><br>

## Step 7: Changing State of 2 Veth Cables 4 Interfaces to Up:
**We need to set the state of both bridge br0 side interface(veth-green-br, veth-red-br) of veth cable to up**
```
sudo ip link set dev veth-green-br up
sudo ip link set dev veth-red-br up
```
<br>

**We need to set the state of both namespace green's and red's side interface(veth-green-ns, veth-red-ns) of veth cable to up**
```
sudo ip netns exec green ip link set dev veth-green-ns up
sudo ip netns exec red ip link set dev veth-red-ns up
```
<br>

**We can check the state of interface successfully turn up or not using below command**
```
sudo ip link list
```
<br><br>

## Step 8: Assign Ip Address and Default Gateway Route in Namespace Interface:
**We need to assign ip address to each interface (veth-green-ns and veth-red-ns) of green and red namespace**
```
sudo ip netns exec green ip addr add 10.0.0.11/24 dev veth-green-ns
sudo ip netns exec red ip addr add 10.0.0.21/24 dev veth-red-ns
```
<br>

**We need to assign default ip route to green and red namespace**
```
sudo ip netns exec green ip route add default via 10.0.0.1
sudo ip netns exec red ip route add default via 10.0.0.1
```
<br><br>

## Step 9: Configure Iptables Firewall Rules:
**We need to add rule for ingoing and outgoing package transmission permission through bridge br0**
```
sudo iptables --append FORWARD --in-interface br0 --jump ACCEPT
sudo iptables --append FORWARD --out-interface br0 --jump ACCEPT
```
<br>

**We need to add postrouting chain rule to allow egress traffic to transmit package between network namespace ( green and red ) and google public ip**
```
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE
```
<br>

**We can check the newly added firewall rules**
```
sudo iptables -t nat -L -n -v
```
<br><br>

## Step 10: Check Connectivity Between Namespaces and Egress Traffic:
**We can test connectivity between red and green namespace by assign tcpdump in green namespace interface and ping this interface from red namespace**
```
sudo ip netns exec green bash
sudo tcpdump -i veth-green-ns
sudo ip netns exec red bash
ping 10.0.0.11
```
<br>

**We can test egress connectivity of red and green namespace with google public ip 8.8.8.8**
```
sudo ip netns exec green bash
ping 8.8.8.8
sudo ip netns exec red bash
ping 8.8.8.8
```

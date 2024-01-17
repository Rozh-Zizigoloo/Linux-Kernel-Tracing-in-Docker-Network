# Linux-Kernel-Tracing-in-Docker-Network
In this project, we intend to use tracing tools to understand the implementation of Docker network modes in Linux and calculate the overhead for each mode. 

## Tools ⚙

- Ubuntu 20+2.04 focal
- Perf
- ftrace
## Docker & tracing 🧩
To do this, we first use perf to trace the stack in order to examine the general behavior of the kernel (in network mode). This can be done by creating a simple netcat or ping. Then, we install Docker and use one of the network modes (bridge, which is the default mode) to perform perf again. This will allow us to understand the difference between the two logs and what functions have been added.

Finally, we use the ftrace tool with function filters to identify the overall performance of Docker and calculate its overhead.
## ftrace without Docker 🔧
For trace, we need a series of filters on kernel functions. Here we filter every function that has the names net, ip, tcp skb.
```
echo 'net*' 'ip*' 'tcp*' 'skb*' >> set_ftrace_filter
cat set_ftrace_filter
echo function_graph >> /sys/kernel/debug/tracing/current_tracer
```
Then,we run trace and get a simple netcat.
```
echo 1 > /sys/kernel/debug/tracing/tracing_on
Tab1 : $nc -l 127.0.0.1 4444
Tab2:$nc -p 9000 127.0.0.0 4444
echo 0 > /sys/kernel/debug/tracing/tracing_on
```
Finally, we save this trace in a file:
```
cat trace > /***/ftrace_without_docker.log
```
![Picture1](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/22d6111f-0e66-433b-95aa-276d272d0fcd)

## Docker(Bridge mode) 🎢
**Docker installation :** 
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
> To create a bridge network mode, in the VM settings section, in the network section, we change its value to bridge.

The output of the ip address show command displays the IP addresses of each network interface (NIC) in the system. Here, there are three network interfaces:
![Screenshot 2024-01-17 165814](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/030c904b-8ca8-440c-ae8b-d225c674b212)

1. Lo: This network interface is related to loopback, which is used for internal system communication.
2. enp0s3: This network interface is related to the internal network, which has the IP address 10.0.2.15. By creating a bridge mode, it is directly connected to the home network and the IP is changed to 172.20.10.10.
3. docker0: This is the virtual network interface used for communication within Docker.
** In fact, by creating a bridge, the host is directly connected to the router, i.e. the IP of the main system.
Docker0:
In fact, after installing docker, a new IP is generated called "docker 0", which is a virtual bridge interface, and all containers communicate with each other and the outside world using this interface, and docker 0 is like a There is a switch that switches between containers.
In this section, all modes show the docker network, and currently we have two default modes, host and bridge.
**Create Container :**
```
sudo docker run -itd --rm -p 8089 --network bridge --name test ubuntu
```
![Picture2](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/57aeddb8-adfc-4346-af59-0cbfad9b4d84)

**docker network inspect bridge :**
```

[
    {
        "Name": "**bridge**",
        "Id": "fcc95430ad74867a739d5b411edc5044395d48df5195523799c631fe1b6f2862",
        "Created": "2024-01-01T11:38:52.68857544+02:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "ce9ba1ba96fda724f9adc3084c00e57466e042edd7484e3c76be90684c481003": {
                "Name": "**test**",
                "EndpointID": "85c0cc7c182babe6aed473c24d0283dd792183ad0c2d509b778628511bcc862a",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "**172.17.0.3/16**",
                "IPv6Address": ""
            },
            "f53dcbbcb8669682da8303e7cabb00a718ba1f8b9265bf42d4d0d756bedc674e": {
                "Name": "**test1**",
                "EndpointID": "f965fb063c60c94f60acba20129188c15d3ec8749914575cd44ab12e371680fc",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```
**making connection :**
To run netcat, you must enter the bash file.

![Picture3](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/03ffe2ae-63a7-492d-9d91-abe2da53553b)

> Ping & NetCat installation:
> ```
>  apt-get install iputils-ping
>  apt-get update -y
>  apt-get install -y netcat
> ```
To test the connection -> ping to enp0s3 (172.20.10.10)

![Picture4](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/0372c634-0906-4ee7-9935-ad8ef2dd8750)

## Bridge mode analysis 🪄

Run netcat from inside container to external network "172.20.10.10" and get perf from this operation.
```
sudo perf record -ae 'net:*,skb:*' --call-graph fp
# 172.20.10.10 ip addresss host
$nc -l 172.20.10.10.8002
$nc -p 9000 172.20.10.10.8082
```
> output file -> trace_with_bridge.txt

![Picture5](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/ee6b3953-d438-4e11-8ceb-ac18208ea4e3)

The checks between the two log files found **bf_forward**:
> File under review:
> trace_simple.txt
> trace_bridge.txt

![Picture6](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/f9a02e87-44bb-47b9-aca5-5677e0bc35c4)

and others functions:

- br_handle_frame
- br_nf_pre_routing
- br_nf_pre_routing_finish
- br_nf_hook_thresh
- br_handle_frame_finish
- br_pass_frame_up
- br_netif_receive_skb
- netif_receive_skb
## Bridge architecture 🏗️

![Picture7](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/22dd8f95-7a37-4208-b4bb-cc71a9ec4000)

**Analysis:**
The implementation of the bridge in Linux is such that the connection connects to the internal kernel.
```
struct net_bridge
{
    spinlock_t          lock;
    struct list_head    port_list;
    struct net_device   *dev;

    spinlock_t          hash_lock;
    struct hlist_head   hash[BR_HASH_SIZE];
    bridge_id           bridge_id;
    ...
}
```
port_list is the list of ports that a bridge has. Each bridge can have a maximum of BR_MAX_PORTS (1024) ports. dev is a pointer to the net_device structure that represents the bridge device. Hash table is sent with BR_HASH_SIZE(256) input.

By following the instructions, we reach this function, which is the main story!

```
int br_add_if(struct net_bridge *br, struct net_device *dev) {
    struct net_bridge_port *p;
    ... (Validation)
    p = new_nbp(br, dev);
    ...
    err = netdev_rx_handler_register(dev, **br_handle_frame**, p);
    ...
    err = netdev_master_upper_dev_link(dev, br->dev);
    ...
    list_add_rcu(&p->list, &br->port_list);
    nbp_update_port_count(br);
    ...
    if (br_fdb_insert(br, p, dev->dev_addr, 0))
        netdev_err(dev, "failed insert local address bridge forwarding table\n");
    ...
    return err;
}
```
Perform a series of validations to ensure that this device can become a bridge slave. Some of the rules are:
1) Non-Ethernet devices are not allowed.
2) A device that is already slave is not allowed.
3) The device itself cannot be a bridge.
  4) IFF_DONT_BRIDGE should not exist on the device. Etc.
Allocate and initialize a **net_bridge_port** structure. The port is added to the port_bridge list later.
Register a **br_handle_frame** receive handler for the device. Frames sent to that device are handled by this function. We'll see what this controller does later.
Make the device a slave, that is, make the bridge the master of this device.
Add the Ethernet address of this device as a local entry to the forwarding table.

The figure below from "Anatomy of a Linux bridge" shows the relationship of the functions we mentioned above.
 
![Picture8](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/d95b5b1f-d2ac-469e-9e96-42fb50803560)

Incoming data is first processed by netif_receive_skb, which is a device-independent public function. rx_handler calls the device on which the data is to be received. Here the br_handle_frame handler is called to process the received data on the device.

br_handle_frame first checks to make sure the frame is a valid Ethernet frame. It then checks to see if the destination is a reserved address, which means it's a control frame. If the answer is positive, special processing is required. Otherwise, it calls br_handle_frame_finish to handle this frame.

```

rx_handler_result_t br_handle_frame(struct sk_buff **pskb) {
    const unsigned char *dest = eth_hdr(skb)->h_dest;
    ... (Validation code)
    if (unlikely(is_link_local_ether_addr(dest))) {
        /*
         * See IEEE 802.1D Table 7-10 Reserved addresses
         *
         * Assignment               Value
         * Bridge Group Address     01-80-C2-00-00-00
         * (MAC Control) 802.3      01-80-C2-00-00-01
         * (Link Aggregation) 802.3 01-80-C2-00-00-02
         * 802.1X PAE address       01-80-C2-00-00-03
         *
         * 802.1AB LLDP         01-80-C2-00-00-0E
         *
         * Others reserved for future standardization
         */
        ... (Special processing for control frame)
    }

    ...
    NF_HOOK(NFPROTO_BRIDGE, NF_BR_PRE_ROUTING, skb, skb->dev, NULL,
            br_handle_frame_finish);
    ...
}

```
> br_handle_frame_finish does the following:
> 
> Get the MAC address of the source and update the forwarding database.

**Analysis with Ftrace :**
```
echo 'br_*' >> set_ftrace_filter
set_ftrace_filter
```
> output file -> bridge_ftrace.log

To use the firefox tool to display the tracing file graphically;
We create a flamegraph file:

![Picture9](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/b246ca8a-fd57-49db-b36b-2e47fd8a2d46)

![Picture10](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/2b0e5295-61bc-41cf-bcbc-02f1322e8941)


![Picture11](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/2af891bf-2261-41d1-b8c4-3116cd0ffba2)


It is shown graphically in the bridge.perf file.
It has an overhead of **19.53**.

## Docker (IPVLAN L2 mode) 🪢

In an IPvlan network, all containers on a Docker host share a single MAC address.

L2 (or layer 2) mode is the default IPvlan mode. In L2 mode, the Docker host acts like a switch between the parent interface and a virtual NIC for each container. All communication is based on MAC addresses only. Containers in an IPvlan network (in L2 mode) can communicate with containers in other IPvlan networks. However, this causes many ARP broadcasts that affect network performance.

In L3 (or Layer 3) mode, the Docker host acts like a Layer 3 device to route packets between the parent interface and the virtual NIC for each container. In this case, containers are completely isolated from other networks and broadcasts are limited to only the Layer 2 subnet. This improves network performance. The only downside is that you have to manually add a static route on your gateway router to tell other network devices how to access your IPvlan network running in L3 mode.

**Ip address show :**
```
enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:cb:20:e8 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/28 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 57880sec preferred_lft 57880sec
    inet6 fe80::2f58:cfa7:6340:43db/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
**create network mode**
```
sudo docker create network ipvlan --subnet 10.0.2.15/24 –gateway 172.20.10.1  --name ipvlanNetwork
```

![Picture12](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/37809eb2-01d0-45b4-a7cb-afb44319713a)

**Create container :**
```
sudo docker run -itd --rm --network ipvlanNetwork --ip 10.0.2.3 --name 1p2vlanCont alpine
```

![Picture13](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/6cf41842-89db-409e-8f09-64f284f4c535)

**making connection :**
To run Ping, you must enter the bash file.

![Picture14](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/ae11fb20-f0d3-4b2f-b2c3-f637c30f2a67)

for trace; We need to ping the eth0 again and get its ip using the following commands.
> ip address show
> ifconfig

![Picture15](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/11caa84c-7f57-4ec0-b966-13208d1064e8)

![Picture16](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/c5fb1450-2123-48be-bcdf-77d465899c30)

## IPVLAN mode analysis 🪄

Run Ping from shell to inside container with  "10.0.2.3" and get perf from this operation.
```
sudo perf record -ae 'net:*,skb:*' --call-graph fp
ping 10.0.2.3
```

![Picture17](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/f0e37014-d15f-4333-b456-1da3951331e2)

> output file -> trace_with_bridge.txt
![Screenshot 2024-01-17 232456](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/5a289395-3f2b-4757-9625-88295ce7ce19)

The checks between the two log files found **netif_rx **:
> File under review:
> trace_simple.txt
> trace_IPVLAN.txt

![Picture6](https://github.com/Rozh-Zizigoloo/Linux-Kernel-Tracing-in-Docker-Network/assets/156912661/f9a02e87-44bb-47b9-aca5-5677e0bc35c4)

and others functions:

-   net:napi_gro_frags_entry
  net:napi_gro_receive_entry
  net:net_dev_queue  
  net:net_dev_start_xmit
  net:net_dev_xmit                                   [Tracepoint eve
  net:netif_receive_skb 
  net:netif_receive_skb_entry  
  net:netif_receive_skb_list_entry 
  net:netif_rx  
  net:netif_rx_entry   
  net:netif_rx_ni_entry

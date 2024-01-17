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
To run netcat, you must enter the bash file; become ubuntu.

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


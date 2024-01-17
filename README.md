# Linux-Kernel-Tracing-in-Docker-Network
In this project, we intend to use tracing tools to understand the implementation of Docker network modes in Linux and calculate the overhead for each mode. 

## Tools ⚙

- Ubuntu 20+2.04 focal
- Perf
- ftrace
## Docker & tracing 🧩
To do this, we first use perf to trace the stack in order to examine the general behavior of the kernel (in network mode). This can be done by creating a simple netcat or ping. Then, we install Docker and use one of the network modes (bridge, which is the default mode) to perform perf again. This will allow us to understand the difference between the two logs and what functions have been added.

Finally, we use the ftrace tool with function filters to identify the overall performance of Docker and calculate its overhead.
## ftrace without Docker :
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

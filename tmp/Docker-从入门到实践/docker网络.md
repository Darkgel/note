# docker网络

命令：docker network

网络驱动：

1. bridge（默认）：通常在各个独立运行的应用容器间需要通信时使用
2. host：对于独立运行的容器，消除容器与docker宿主机之间的网络隔离，直接使用宿主机的网络
3. overlay：连接多个docker守护进程，使得swarm services之间可以通信
4. macvlan：允许将MAC地址赋值给容器，使得容器仿佛是网络的物理设备
5. none：使得容器无法使用网络
6. Network plugins

Network driver summary

- User-defined bridge networks are best when you need multiple containers to communicate on the same Docker host.
- Host networks are best when the network stack should not be isolated from the Docker host, but you want other aspects of the container to be isolated.
- Overlay networks are best when you need containers running on different Docker hosts to communicate, or when multiple applications work together using swarm services.
- Macvlan networks are best when you are migrating from a VM setup or need your containers to look like physical hosts on your network, each with a unique MAC address.
- Third-party network plugins allow you to integrate Docker with specialized network stacks.

## bridge networks

In terms of Docker, a bridge network uses a software bridge which allows containers connected to the same bridge network to communicate, while providing isolation from containers which are not connected to that bridge network.

containers on different bridge networks cannot communicate directly with each other

User-defined bridges provide automatic DNS resolution between containers.Containers on the default bridge network can only access each other by IP addresses, unless you use the --link option, which is considered legacy. On a user-defined bridge network, containers can resolve each other by name or alias.

比较default bridge和user-defined bridge

Containers connected to the same user-defined bridge network effectively expose all ports to each other. For a port to be accessible to containers or non-Docker hosts on different networks, that port must be published using the -p or --publish flag.

Containers connected to the default bridge network can communicate, but only by IP address, unless they are linked using the legacy --link flag.

### tutorial

default bridge network

## overlay networks

The overlay network driver creates a distributed network among multiple Docker daemon hosts. This network sits on top of (overlays) the host-specific networks, allowing containers connected to it (including swarm service containers) to communicate securely.

## host networking

当使用host networking运行容器时，容器不会被分配ip，而是直接使用host的ip。此时端口映射将无法使用(-p,--publish,-P 和 --publish-all 选项将被忽略)

host networking可以提高性能

the host networking driver only works on Linux hosts

## macvlan networks

## 不使用网络

只能使用loopback
docker run --rm -dit \
  --network none \
  --name no-net-alpine \
  alpine:latest \
  ash

一个容器可以同时处于多个网络中

## 其他

在linux中，docker通过iptables rule来提供网络隔离

### Container networking

#### Published ports

默认情况下，当你创建一个容器的时候，该容器是不会将它的端口公开给（publish）外部世界的。为了使得docker外的服务，或者是那些不连接在同一个网络中的容器可以访问到该端口，需要在运行容器时使用-p（--publish）选项。通过使用该选项，会创建一些从容器端口到宿主机端口的端口映射。

- -p 8080:80	Map TCP port 80 in the container to port 8080 on the Docker host.
- -p 192.168.1.100:8080:80	Map TCP port 80 in the container to port 8080 on the Docker host for connections to host IP 192.168.1.100.
- -p 8080:80/udp	Map UDP port 80 in the container to port 8080 on the Docker host.
- -p 8080:80/tcp -p 8080:80/udp	Map TCP port 80 in the container to TCP port 8080 on the Docker host, and map UDP port 80 in the container to UDP port 8080 on the Docker host.

#### IP address and hostname

默认情况下容器都会分配到其所在网络中的一个ip，此时docker daemon的作用类似于DHCP服务器。

#### DNS services

默认情况下，容器会继承docker daemon的dns配置，例如/etc/hosts/和/etc/resolv.conf。而外配置：

- --dns	The IP address of a DNS server. To specify multiple DNS servers, use multiple --dns flags. If the container cannot reach any of the IP addresses you specify, Google’s public DNS server 8.8.8.8 is added, so that your container can resolve internet domains.
- --dns-search	A DNS search domain to search non-fully-qualified hostnames. To specify multiple DNS search prefixes, use multiple --dns-search flags.
- --dns-opt	A key-value pair representing a DNS option and its value. See your operating system’s documentation for resolv.conf for valid options.
- --hostname	The hostname a container uses for itself. Defaults to the container’s ID if not specified.

添加：https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
# Docker installation and networking setup

*Docker 1.10 is required.*

## Installation

To install latest available RC version:

```
curl -fsSL https://test.docker.com/ | sh
```

This will fail on ubuntu-xenial, but wily packages are working.

1. fix the repository:

  ```
  $ cat /etc/apt/sources.list.d/docker.list
  deb [arch=amd64] https://apt.dockerproject.org/repo ubuntu-wily testing
  ```

1. install Docker

  ```
  sudo apt-get update
  sudo apt-get install docker-engine
  ```

1. run the service

  ```
  sudo systemctl enable docker.service
  sudo systemctl start docker.service
  ```

## Networking configuration

https://docs.docker.com/engine/userguide/networking/dockernetworks/

How to setup a container with two network interfaces:

1. create the second network (the first one is the default `bridge0`):

  ```
  sudo docker network create --driver bridge --subnet 10.0.1.0/24 isolated_nw
  ```

1. create container using the default network:

  ```
  sudo docker create -it --name node1 fedora:20 /bin/bash
  ```

1. connect the container to the second network:

  ```
  sudo docker network connect --ip=10.0.1.2 isolated_nw node1
  ```

1. start the container and check network interfaces:

  ```
  sudo docker start -i node1
  [root@2fd6835273b2 /]# ip addr
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group   default
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
         valid_lft forever preferred_lft forever
  16: eth0@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue   state UP group default
      link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.2/16 scope global eth0
         valid_lft forever preferred_lft forever
      inet6 fe80::42:acff:fe11:2/64 scope link
         valid_lft forever preferred_lft forever
  18: eth1@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue   state UP group default
      link/ether 02:42:0a:00:01:02 brd ff:ff:ff:ff:ff:ff
      inet 10.0.1.2/24 scope global eth1
         valid_lft forever preferred_lft forever
      inet6 fe80::42:aff:fe00:102/64 scope link
         valid_lft forever preferred_lft forever
  ```

If all you need is static addressing with a single network, you may setup
networking during container creation:

```sudo docker run -it --net=isolated_nw --name=node1 --ip=10.0.1.2 fedora:20 /bin/bash
```

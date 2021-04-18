# Networking

## Virtual Networks

- Each container is connected to a private virtual network, `bridge` by default
- Each network routes out through NAT firewall on host IP
- Containers on the same virtual network don't need `-p` to talk to each other
  - Containers on the same app should be on the same network
- Can skip the virtual network and use host IP with `--net=host` if necessary

### Show the port mapping for a container

```docker
> docker container run -p 80:80 --name webhost -d nginx
a409fe40ae853222275e5fdc45b026496e75ff7ba49d38ebc576e02dfe3b43ba
> docker container port webhost
80/tcp -> 0.0.0.0:80
```

### By default, the IP address of the container is not the same as the host

```docker
> docker container inspect --format "{{ .NetworkSettings.IPAddress}}" webhost
172.17.0.4
```

### List the docker netowrks

```docker
> docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
fbba7bae184b   bridge    bridge    local
8eb3be870a8b   host      host      local
c734337a9b37   none      null      local
```

### See what containers are connected to a network

```docker
> docker network inspect bridge
...
        "Containers": {
            "a409fe40ae853222275e5fdc45b026496e75ff7ba49d38ebc576e02dfe3b43ba": {
                "Name": "webhost",
                "EndpointID": "6c2bfb377cb7637a487f73d4c2f1d180759952b43550346ec4fecd6249ef534b",
                "MacAddress": "02:42:ac:11:00:04",
                "IPv4Address": "172.17.0.4/16",
                "IPv6Address": ""
            },
...
```

### Create a new virtual network

The default driver is `bridge`

```docker
> docker network create my_app_net
f2337102e4a2088dda27358d709f3df5a0b68a6bb1db81fc9ed236f2054c42dd
> docker network ls
NETWORK ID     NAME         DRIVER    SCOPE
fbba7bae184b   bridge       bridge    local
8eb3be870a8b   host         host      local
f2337102e4a2   my_app_net   bridge    local
c734337a9b37   none         null      local
```

### Create a new docker container on a specific network

```docker
> docker container run -d --name new_nginx --network my_app_net nginx
86e6612fe9651c1c4d2c0d604fd52be0bf92f987c23b6fea7ede1148db4e7af8
> docker network inspect my_app_net    
...
        "Containers": {
            "86e6612fe9651c1c4d2c0d604fd52be0bf92f987c23b6fea7ede1148db4e7af8": {
                "Name": "new_nginx",
                "EndpointID": "bb2744d501d68ea328e21d6e2e96b7e7e742ead6d0d41e39b91975e46b6c3c00",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
...
```

### Connect an existing container to and existing network

```docker
> docker network ls         
NETWORK ID     NAME         DRIVER    SCOPE
...
f2337102e4a2   my_app_net   bridge    local
...
> docker container ls
CONTAINER ID   IMAGE     COMMAND                  CREATED             STATUS             PORTS                 NAMES    
...
a409fe40ae85   nginx     "/docker-entrypoint.…"   24 minutes ago      Up 24 minutes      0.0.0.0:80->80/tcp    webhost  
...
> docker network connect f2337102e4a2 a409fe40ae85
> docker container inspect a409fe40ae85
...
            "Networks": {
                "bridge": {
...
                 },
                "my_app_net": {
...
```

### Disconnect an existing container from a network

```docker
> docker network disconnect f2337102e4a2 a409fe40ae85
> docker container inspect a409fe40ae85
...
            "Networks": {
                "bridge": {
...
```

## DNS

- Docker container names are used as hostnames on the network
- The default `bridge` network does not include DNS out of the box

### Ping one container from bash on another (Default alpine does not come with ping installed)

```docker
> docker container run -d --name my_nginx --network my_app_net nginx
0cfdda74876b5080a8c886b395a79e0278451e26a587cd508098178df6c31858
> docker network inspect f2337102e4a2
...
        "Containers": {
            "0cfdda74876b5080a8c886b395a79e0278451e26a587cd508098178df6c31858": {
                "Name": "my_nginx",
...
            },
            "86e6612fe9651c1c4d2c0d604fd52be0bf92f987c23b6fea7ede1148db4e7af8": {
                "Name": "new_nginx",
...
PS C:\repos\udemy_docker_mastery> docker container exec -it my_nginx bash
root@0cfdda74876b:/# apt-get install ping
Reading package lists... Done
...
Processing triggers for libc-bin (2.28-10) ...
root@0cfdda74876b:/# exit
exit
PS C:\repos\udemy_docker_mastery> docker container exec -it my_nginx ping new_nginx
PING new_nginx (172.18.0.2) 56(84) bytes of data.
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=1 ttl=64 time=0.082 ms
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=2 ttl=64 time=0.112 ms
...
```

## CLI App Testing

- Use the `--rm` flag to remove the container after the process ends

### Check curl version on centos:7

```docker 
> docker container run --rm -it --name centos centos:7 bash
[root@27a2e762db44 /]# yum update curl
Loaded plugins: fastestmirror, ovl
...
Complete!
[root@27a2e762db44 /]# curl --version
curl 7.29.0 (x86_64-redhat-linux-gnu) libcurl/7.29.0 NSS/3.53.1 zlib/1.2.7 libidn/1.28 libssh2/1.8.0
...
[root@27a2e762db44 /]# exit
exit
```

### Check cURL version on ubuntu:14.04

```docker
> docker container run --rm -it --name ubuntu ubuntu:14.04 bash
root@ff54bd76ad3d:/# apt-get update && apt-get install curl
Get:1 http://security.ubuntu.com trusty-security InRelease [65.9 kB]
...
Processing triggers for libc-bin (2.19-0ubuntu6.15) ...
root@ff54bd76ad3d:/# curl --version
curl 7.35.0 (x86_64-pc-linux-gnu) libcurl/7.35.0 OpenSSL/1.0.1f zlib/1.2.8 libidn/1.28 librtmp/2.3
...
root@ff54bd76ad3d:/# exit
exit
```

## DNS Round Robin ("Poor man's load balancer" - Multiple containers behind the same domain name)

- Create a network `dude`
- Put two new instances of the elasticsearch:2 image ont he network
  - Both of them have the network alias `search`
- Run `nslookup` on an instance of `alpine` to resolve the IP address of the domain name `search`, showing that there are two IP addresses at the same domain name
- Run `curl -s search:9200` on an instance of `centos` to get info about the container at the domain name `search`. Perform this multiple times to show how the load is distributed between the two containers.

```docker
docker network create dude
a2532907dc9872181854ecd524e477da793496e3128576890261af4d612a7761
> docker run -d --net dude --net-alias search elasticsearch:2
Unable to find image 'elasticsearch:2' locally
...
61814c4f1f502a2f06e8591d0ad101a3fee60169847939e134c5518310167e7f
PS C:\repos\udemy_docker_mastery> docker run -d --net dude --net-alias search elasticsearch:2
2cd439f48c7ff22527f0feff53ff99b0fecdca6ad69fc3026d8623f07e9a8e3a
> docker container ls
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS                NAMES
2cd439f48c7f   elasticsearch:2   "/docker-entrypoint.…"   36 seconds ago   Up 34 seconds   9200/tcp, 9300/tcp   boring_moore   
61814c4f1f50   elasticsearch:2   "/docker-entrypoint.…"   45 seconds ago   Up 42 seconds   9200/tcp, 9300/tcp   ecstatic_panini
> docker container run --rm --net dude alpine nslookup search
Server:         127.0.0.11
Address:        127.0.0.11:53

Non-authoritative answer:
*** Cant find search: No answer

Non-authoritative answer:
Name:   search
Address: 172.19.0.2
Name:   search
Address: 172.19.0.3
> docker container run --rm --net dude centos curl -s search:9200
Unable to find image 'centos:latest' locally
latest: Pulling from library/centos
7a0437f04f83: Pull complete
Digest: sha256:5528e8b1b1719d34604c87e11dcd1c0a20bedf46e83b5632cdeac91b8c04efc1
Status: Downloaded newer image for centos:latest
{
  "name" : "Saint Elmo",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "j3EV_9tbTau551b-Ty_OWA",
  "version" : {
    "number" : "2.4.6",
    "build_hash" : "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp" : "2017-07-18T12:17:44Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.4"
  },
  "tagline" : "You Know, for Search"
}
> docker container run --rm --net dude centos curl -s search:9200
{
  "name" : "Omen",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "wVfE_38VS4i_AMb-8LGHhw",
  "version" : {
    "number" : "2.4.6",
    "build_hash" : "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp" : "2017-07-18T12:17:44Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.4"
  },
  "tagline" : "You Know, for Search"
}
```

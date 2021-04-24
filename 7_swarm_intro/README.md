
# Docker Swarm

- How to automate container lifecycle?
- How to easily scale up/down
- Ensure containers are re-created if they fail

Swam Mode
- A server clustering solution
- Worker/manager topology
  - Managers in the Raft consensus group
  - Workers in the gossip group

# Set Up

By default, swarm features are disabled.

```s
> docker info
Client:
...
Server:
...
 Swarm: inactive
...
```

To enable, just call init

```s
$ docker swarm init
```

## Swarm Hosts

- play-with-docker.com
  - Resets agter 4 hours
- `docker-machine` + VirtualBos
  - Free and runs locally
- docker-machine to provision AWS/Azure/DO/Google
  - Not designed for production, so cloud platform setup not worth the effort
- Digital Ocean + Docker local
  - Most like production
  - Nodes cost money

# Using Digital Ocean

## Create nodes on Digital Ocean and record the ip addresses

https://docs.digitalocean.com/products/getting-started/

## Create a local ssh config to avoid typing in IP addresses

Create the config file if you haven't already

```s
$ touch ~/.ssh/config
$ vi ~/.ssh/config
```

Insert the contents

```config
Host node1
    HostName 128.199.7.46
    User root

Host node2
    HostName 143.198.230.146
    User root

Host node3
    HostName 143.198.233.146
    User root
```

Then just ssh into the host and install docker

```s
$ ssh node1
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
```

Open Docker Swarm ports on each machine

- TODO

### Node 1

```s
$ docker swarm init --advertise-addr 128.199.7.46
```

Copy the command printed to add a worker to the swarm

### Node 2

```s
$ docker swarm join --token SWMTKN-1-5ho4c8a8eygfuzaqkqa79tee61zlbki1zoc85vw9wzzepqjink-dxjslrhds93hyhqbh81tucsxs 128.199.7.46:2377
```

### Node 1

```s
$ docker node ls
b821dky3tqa770i0z1yf558nh *   node1      Ready     Active         Leader           20.10.6
a5zh7dgx9kp0b86goce9lzqkt     node2      Ready     Active                          20.10.6
```

Node 1 is now a manager and Node 2 is a worker

Upgrade Node 2 to a manager

```s
$ docker node update --role manager node2
b821dky3tqa770i0z1yf558nh *   node1      Ready     Active         Leader           20.10.6
a5zh7dgx9kp0b86goce9lzqkt     node2      Ready     Active         Reachable        20.10.6
```

Get the token to add a node as a manager by default

```s
$ docker swarm join-token worker
docker swarm join --token SWMTKN-1-5ho4c8a8eygfuzaqkqa79tee61zlbki1zoc85vw9wzzepqjink-cai7j2xt3otq69t3jl8ivapkg 128.199.7.46:2377
```

### Node 3 

```s
$ docker swarm join --token SWMTKN-1-5ho4c8a8eygfuzaqkqa79tee61zlbki1zoc85vw9wzzepqjink-cai7j2xt3otq69t3jl8ivapkg 128.199.7.46:2377
This node joined a swarm as a manager.
```

### Node 1

Confirm the 3 swarm managers

```s
$ docker node ls
b821dky3tqa770i0z1yf558nh *   node1      Ready     Active         Leader           20.10.6
a5zh7dgx9kp0b86goce9lzqkt     node2      Ready     Active         Reachable        20.10.6
qo2nfka8e4wzk9a6qvpus0r14     node3      Ready     Active         Reachable        20.10.6
```

Create a new service with 3 replicas

```s
$ docker service create --replicas 3 alpine ping 8.8.8.8
$ docker service ls
ID             NAME          MODE         REPLICAS   IMAGE           PORTS
igt29a5dv2m9   gracious_tu   replicated   3/3        alpine:latest
```

List the service on this node

```s
$ docker node ps
```

List the service on another node

```s
$ docker node ps node2
```

List the service on all nodes
List the service `gracious_tu` on all nodes

```s
$ docker service ps gracious_tu
```

- Remove the service

```s
$ docker service rm gracious_tu
```

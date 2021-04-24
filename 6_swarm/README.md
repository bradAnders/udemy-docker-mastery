
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

 # Multi-Host Networking

 - Choose `--driver overlay` when creating a network


### Node 1

- Create a new overlay network `mydrupal`

 ```s
$ docker network create --driver overlay mydrupal
2dnbywgvftb3rpufz2y6jmmcb
$ docker network ls
NETWORK ID     NAME              DRIVER    SCOPE
1be4aaa58aef   bridge            bridge    local
4f2b66916dc8   docker_gwbridge   bridge    local
388b541f5e20   host              host      local
5ax80b9eqp3i   ingress           overlay   swarm
2dnbywgvftb3   mydrupal          overlay   swarm
f8f805bf9abb   none              null      local
 ```

 - Create a postgres service and add it to the network

```s
$ docker service create --name psql --network mydrupal -e POSTGRES_PASSWORD=mypass postgres
$ docker service ls
ID             NAME          MODE         REPLICAS   IMAGE             PORTS  
n6kjpk0lx9jj   psql          replicated   1/1        postgres:latest 
```
- Services run through the orchestrator/scheduler
  - Cannot run them in the foreground
  - Will not see image download outputs to shell

```s
$ docker service ps psql
ID             NAME      IMAGE             NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
aghi0wwfl1lt   psql.1    postgres:latest   node1     Running         Running 2 minutes ago  
```

- Instead, inspect the logs to check if it's running

```s
$ docker service logs psql
psql.1.aghi0wwfl1lt@node1    | The files belonging to this database system will be owned by user "postgres".
psql.1.aghi0wwfl1lt@node1    | This user must also own the server process.
...
psql.1.aghi0wwfl1lt@node1    | 
psql.1.aghi0wwfl1lt@node1    | PostgreSQL init process complete; ready for start up.
psql.1.aghi0wwfl1lt@node1    | 
psql.1.aghi0wwfl1lt@node1    | 2021-04-24 15:18:09.011 UTC [1] LOG:  starting PostgreSQL 13.2 (Debian 13.2-1.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
psql.1.aghi0wwfl1lt@node1    | 2021-04-24 15:18:09.012 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
psql.1.aghi0wwfl1lt@node1    | 2021-04-24 15:18:09.014 UTC [1] LOG:  listening on IPv6 address "::", port 5432
psql.1.aghi0wwfl1lt@node1    | 2021-04-24 15:18:09.017 UTC [1] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
psql.1.aghi0wwfl1lt@node1    | 2021-04-24 15:18:09.022 UTC [65] LOG:  database system was shut down at 2021-04-24 15:18:08 UTC
psql.1.aghi0wwfl1lt@node1    | 2021-04-24 15:18:09.028 UTC [1] LOG:  database system is ready to accept connections
```

- Create the drupal service

```s
$ docker service create --name drupal --network mydrupal -p 80:80 drupal
```

- Watch the output of `docker serivce ls` until the service comes online

```s
$ watch docker service ls
```

- Note that the drupal service is running on a different node

```s
$ docker service ps drupal
ID             NAME       IMAGE           NODE      DESIRED STATE   CURRENT STATE                ERROR     PORTS
njnlcr65fv72   drupal.1   drupal:latest   node2     Running         Running about a minute ago 
```

### Web setup

- Navigate to the IP address of node2 in the browser (http://143.198.230.146)
- Setup the drupal site as before
- For the database hostname, use the service name `psql`
  - Database: `postgres`
  - User: `postgres`
  - Password: `mypass`
  - Hostname: `psql`

Note that the HTTP server is running on `node2`, but the website is accessible from all node IP addresses
- `node1`: http://128.199.7.46
- `node2`: http://143.198.230.146
- `node3`: http://143.198.233.146

The `drupal` serivce only has on IP address on the overlay network `2dnbywgvftb3`

```s
$ docker service inspect drupal
...
            "VirtualIPs": [
...
                {
                    "NetworkID": "2dnbywgvftb3rpufz2y6jmmcb",
                    "Addr": "10.0.1.5/24"
                }
            ]
...
```

This is because of the Routing Mesh

### Routing Mesh

- Distributes ingress packets for a service to a proper task
- Spans all nodes in a swarm
- Uses IPVS from Linux Kernel
- Listening and load balancing on all nodes
  - Container to container in an `overlay` network (uses Virtual IP)
  - External traffic to published ports (All nodes listen to each service's ports)

### Example

- Create an `elasticsearch` service

```s
$ docker service create --name search --replicas 3 -p 9200:9200 elasticsearch:2
```

- Check that it's running

```s
$ docker service ps search
```

- Test it out

```s
$ curl localhost:9200
{
  "name" : "Batragon",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "ZGejQT8_R82G8aINIWyfvg",
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

- Load balancing is stateless
  - Won't work with session cookies without additional configuration (Such as nginx or HAProxy)
  - OSI layer 3 (TCP), not layer 4 (DNS)


# Multi-Service App Example




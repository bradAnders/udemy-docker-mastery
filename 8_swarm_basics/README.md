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


### Networks

```s
$ docker network create --driver overlay frontend
$ docker network create --driver overlay backend
```



### Vote service

```s
$ docker service create -p 80:80 --network frontend --replicas 2 --name vote bretfisher/examplevotingapp_vote
```

### Redis service

```s
$ docker service create --network frontend --replicas 1 --name redis redis:3.2
```

### Worker service

```s
$ docker service create --network frontend --network backend --replicas 1 --name worker bretfisher/examplevotingapp_worker:java 
```

### Database service

```s
$ docker service create --mount type=volume,source=db-data,target=/var/lib/postgresql/data --network backend --replicas 1 -e POSTGRES_HOST_AUTH_METHOD=trust --name db postgres:9.4 
```

### Result service

```s
$ docker service create -p 5001:80 --network backend --replicas 1 --name result bretfisher/examplevotingapp_result
```

# Stacks: Production Grade Compose


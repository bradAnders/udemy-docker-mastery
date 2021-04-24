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

## Web setup

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

## Routing Mesh

- Distributes ingress packets for a service to a proper task
- Spans all nodes in a swarm
- Uses IPVS from Linux Kernel
- Listening and load balancing on all nodes
  - Container to container in an `overlay` network (uses Virtual IP)
  - External traffic to published ports (All nodes listen to each service's ports)

## Example

### Create an `elasticsearch` service

```s
$ docker service create --name search --replicas 3 -p 9200:9200 elasticsearch:2
```

### Check that it's running

```s
$ docker service ps search
```

### Test it out

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

### Copy the compose file to `node1` and verify success

```s
$ cd udemy-docker-mastery/8_swarm_basics/swarm-stack-1
~/udemy-docker-mastery/8_swarm_basics/swarm-stack-1 $ scp example-voting-app-stack.yml node1:/root
~/udemy-docker-mastery/8_swarm_basics/swarm-stack-1 $ ssh node1
$ ls -la
```

### Deploy the app

```s
$ docker stack deploy -c example-voting-app-stack.yml voteapp
```

### List the stacks

```s
$ docker stack ls
NAME      SERVICES   ORCHESTRATOR
voteapp   6          Swarm
```

### List the tasks in a stack

```s
$ docker stack ps voteapp 
ID             NAME                   IMAGE                                       NODE      DESIRED STATE   CURRENT STATE                ERROR     PORTS
vmso52zs8bsn   voteapp_db.1           postgres:9.4                                node1     Running         Running about a minute ago             
pswa2kjnstn5   voteapp_redis.1        redis:alpine                                node3     Running         Running about a minute ago             
0f0rtv1tzn8l   voteapp_result.1       bretfisher/examplevotingapp_result:latest   node3     Running         Running about a minute ago             
vy7kz1qze4mu   voteapp_visualizer.1   dockersamples/visualizer:latest             node2     Running         Running 42 seconds ago                 
q35zuwyd688m   voteapp_vote.1         bretfisher/examplevotingapp_vote:latest     node2     Running         Running about a minute ago             
xottmbtpwjry   voteapp_vote.2         bretfisher/examplevotingapp_vote:latest     node1     Running         Running about a minute ago             
zz5biqgkppyq   voteapp_worker.1       bretfisher/examplevotingapp_worker:java     node1     Running         Running about a minute ago 
```

### Note that the tasks are different than the containers

- Each container has a GUID

```s
$ docker container ls
CONTAINER ID   IMAGE                                     COMMAND                  CREATED         STATUS         PORTS      NAMES
ca24de46e692   postgres:9.4                              "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   5432/tcp   voteapp_db.1.vmso52zs8bsnr2crlrsod4r41
9dd5c176e138   bretfisher/examplevotingapp_worker:java   "java -XX:+UseContai…"   4 minutes ago   Up 4 minutes              voteapp_worker.1.zz5biqgkppyq9cgd7j87yok2f
57252b3a6ec9   bretfisher/examplevotingapp_vote:latest   "gunicorn app:app -b…"   4 minutes ago   Up 4 minutes   80/tcp     voteapp_vote.2.xottmbtpwjryun6qswbpuptnr
```

### Show the services in the stack

```s
$ docker stack services voteapp 
ID             NAME                 MODE         REPLICAS   IMAGE                                       PORTS
vwje2zxza6mr   voteapp_db           replicated   1/1        postgres:9.4                                
ohu308ujtec2   voteapp_redis        replicated   1/1        redis:alpine                                
oydn7dqchtup   voteapp_result       replicated   1/1        bretfisher/examplevotingapp_result:latest   *:5001->80/tcp
sq9zmx6jkgap   voteapp_visualizer   replicated   1/1        dockersamples/visualizer:latest             *:8080->8080/tcp
s8gr7v0w91rn   voteapp_vote         replicated   2/2        bretfisher/examplevotingapp_vote:latest     *:5000->80/tcp
shlaayy8ywdu   voteapp_worker       replicated   1/1        bretfisher/examplevotingapp_worker:java      
```

### Visit the sites

- `vote`: http://128.199.7.46:5000
- `results`: http://128.199.7.46:5001
- `visualizer`: http://128.199.7.46:8080

### Change the compose file and update the service

```s
$ vi example-voting-app-stack.yml
...
$ docker stack deploy -c example-voting-app-stack.yml voteapp 
```

# Swarm Secrets

### Copy the `secrets-sample-1` directory to a node and verify success

```s
$ cd udemy-docker-mastery/8_swarm_basics
~/udemy-docker-mastery/8_swarm_basics $ scp -r secrets-sample-1/ node1:/root/secrets-sample-1
~/udemy-docker-mastery/8_swarm_basics $ ssh node1
$ cd secrets-sample-1
~/secrets-sample-1 $ cat psql_user.txt
mypsqluser
```

### Create a swarm secret from a file

```s
$ docker secret create psql_user psql_user.txt
zv0iwumvz7rd6n19coqg5qamk
```

- DRAWBACK: Storing the password on the host's disk
  - Would prefer a remote API for this

### Create a swarm secret from the command line
```s
$ echo "myDBpassWORD" | docker secret create psql_pass -
1xnd2vsf8izn1b1b784gnlnol
```

- DRAWBACK: This is being stored in the bash history for root

### Inspect meta data for the secrets

```s
$ docker secret inspect psql_pass
```

### Create a service and give it access to the secret

`/run/secrets` is an in-memory file system- secrets are not stored on disk

```s
$ docker service create --name psql --secret psql_user --secret psql_pass -e POSTGRES_PASSWORD_FILE=/run/secrets/psql_pass -e POSTGRES_USER_FILE=/run/secrets/psql_user postgres
```

### Find out which node the container is on

- Containers are only visible from the node they are on

```s
$ docker service ps psql
ID             NAME      IMAGE             NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
wjhto3ca8av1   psql.1    postgres:latest   node3     Running         Running 3 minutes ago  
$ docker exec -it p
```

### Remote into the node and inspect the secret

```s
~/udemy-docker-mastery/8_swarm_basics $ ssh node3
$ docker container ls
CONTAINER ID   IMAGE             COMMAND                  CREATED         STATUS         PORTS      NAMES
61161b6b3535   postgres:latest   "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   5432/tcp   psql.1.dqy903z95aystl7uwfwhk8pao
$ docker exec -it psql.1.dqy903z95aystl7uwfwhk8pao bash
61161b6b3535:/ $ ls /run/secrets
psql_pass  psql_user
61161b6b3535:/ $ cat /run/secrets/psql_pass 
myDBpassWORD
```

### Secrets can be added and removed, but this re-deploys the container (undesired)

```s
$ docker service update --secret-rm psql_pass psql
$ docker service update --secret-add source=psql_pass,target=psql_pass psql
```

# Secrets in Stacks

- Secrets in stacks require compose version >= `"3.1"`

```s
~/udemy-docker-mastery/8_swarm_basics $ scp -r secrets-sample-2 node1:/root/secrets-sample-2
~/udemy-docker-mastery/8_swarm_basics $ ssh node1
$ cd secrets-sample-2
~/secrets-sample-2 $ cat docker-compose.yml
```

### Notice the new `secrets` patterns

```yaml
version: "3.1"

services:
  psql:
    image: postgres
    secrets:
      - psql_user
      - psql_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/psql_password
      POSTGRES_USER_FILE: /run/secrets/psql_user

secrets:
  psql_user:
    file: ./psql_user.txt
  psql_password:
    file: ./psql_password.txt

```

### Deploy

```s
~/secrets-sample-2 $ docker stack deploy -c docker-compose.yml mydb
~/secrets-sample-2 $ docker secret ls
ID                          NAME                 DRIVER    CREATED              UPDATED
y4qh3en8pnrwghd9z9ut73apa   mydb_psql_password             About a minute ago   About a minute ago
4nfie2puvwy0mzy3w62t6hm7y   mydb_psql_user                 About a minute ago   About a minute ago
```

### Stopping the service now removes the secrets as well

```s
~/secrets-sample-2 $ docker stack rm mydb 
Removing service mydb_psql
Removing secret mydb_psql_user
Removing secret mydb_psql_password
Removing network mydb_default
root@node1:~/secrets-sample-2 $ docker secret ls
ID        NAME      DRIVER    CREATED   UPDATED 
```

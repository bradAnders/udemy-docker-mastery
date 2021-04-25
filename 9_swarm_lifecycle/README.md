# Secrets in Compose (Local Development)

- Note that in local development, stacks and secrets are not available

```s
$ docker node ls
Error response from daemon: This node is not a swarm manager. Use "docker swarm init" or "docker swarm join" to connect this node to swarm and try again.
```

### Start up a service using `docker-compose`

- Secrets are available with `docker-compose`, but they are not secure
- This functionality is available only for development purposes

```s
$ cd ~/udemy-docker-mastery/8_swarm_basics/secrets-sample-2
$ docker-compose up -d
$ docker-compose exec psql cat /run/secrets/psql_user
dbuser
```

# Compose Overrides

- Notice the several `docker-compose` files in this directory
- The base `docker-compose.yml` is inherited in all of them
- The implementation `docker-compose.override.yml` is used by default without the `-c` argment with the `docker-compose` CLI
  - Therefore `docker-compose.override.yml` is most suitable for local usage 

```s
$ cd ~/udemy-docker-mastery/9_swarm_lifecycle/swarm-stack-3
$ ls -a
total 28
-rw-r--r-- 1 pi pi  582 Apr 18 04:46 docker-compose.override.yml
-rw-r--r-- 1 pi pi  558 Apr 18 04:46 docker-compose.prod.yml
-rw-r--r-- 1 pi pi  638 Apr 18 04:46 docker-compose.test.yml
-rw-r--r-- 1 pi pi  107 Apr 18 04:46 docker-compose.yml
-rw-r--r-- 1 pi pi  520 Apr 18 04:46 Dockerfile
-rw-r--r-- 1 pi pi    9 Apr 18 04:46 psql-fake-password.txt
drwxr-xr-x 2 pi pi 4096 Apr 18 04:46 themes
```

### Start up the development file and inspect it

- Notice the mounts are created from the `.override.yml` file

```s
$ docker-compose up -d
$ docker inspect swarm-stack-3_drupal_1
...
        "Mounts": [
            {
                "Type": "volume",
                "Name": "swarm-stack-3_drupal-sites",
                "Source": "/var/lib/docker/volumes/swarm-stack-3_drupal-sites/_data",
                "Destination": "/var/www/html/sites",
                "Driver": "local",
                "Mode": "rw",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "volume",
                "Name": "swarm-stack-3_drupal-profiles",
                "Source": "/var/lib/docker/volumes/swarm-stack-3_drupal-profiles/_data",
                "Destination": "/var/www/html/profiles",
                "Driver": "local",
                "Mode": "rw",
                "RW": true,
                "Propagation": ""
            },
            {
                "Type": "bind",
                "Source": "/home/pi/udemy-docker-mastery/9_swarm_lifecycle/swarm-stack-3/themes",
                "Destination": "/var/www/html/themes",
                "Mode": "rw",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "volume",
                "Name": "swarm-stack-3_drupal-modules",
                "Source": "/var/lib/docker/volumes/swarm-stack-3_drupal-modules/_data",
                "Destination": "/var/www/html/modules",
                "Driver": "local",
                "Mode": "rw",
                "RW": true,
                "Propagation": ""
            }
        ],
...
```

### Example CI Command

- The base `.yml` file is called first, and the overlays are called with an additional `-f` argument
- Notice there are no bind mounts upon inspection
  - We don't care about persistent data in CI tests

```s
$ docker-compose -f docker-compose.yml -f docker-compose.test.yml up -d
$ docker inspect swarm-stack-3_drupal_1
```

### Generate an Equivalent Production Command

```s
$ docker-compose -f docker-compose.yml -f docker-compose.prod.yml config > output.yml
secrets:
  psql-pw:
    external: true
services:
  drupal:
    image: custom-drupal:latest
    ports:
    - 80:80/tcp
    volumes:
    - drupal-modules:/var/www/html/modules:rw
    - drupal-profiles:/var/www/html/profiles:rw
    - drupal-sites:/var/www/html/sites:rw
    - drupal-themes:/var/www/html/themes:rw
  postgres:
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/psql-pw
    image: postgres:12.1
    secrets:
    - source: psql-pw
    volumes:
    - drupal-data:/var/lib/postgresql/data:rw
version: '3.1'
volumes:
  drupal-data: {}
  drupal-modules: {}
  drupal-profiles: {}
  drupal-sites: {}
  drupal-themes: {}
```

# Service Updates

### Create a service

```s
$ ssh node1
$ docker service create -p 8088:80 --name web nginx:1.13.7
$ docker service ls
ID             NAME      MODE         REPLICAS   IMAGE          PORTS
v68p5reboacr   web       replicated   1/1        nginx:1.13.7   *:8088->80/tcp
```

### Scale the web service

```s
$ docker service scale web=5
web scaled to 5
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
```

### Change the image

```s
$ docker service update --image nginx:1.13.6 web
```

### Change the published port

```s
$ docker service update --publish-rm 8088 --publish-add 9090:80 web
```

### Forcing an update of a service performs a rebalancing

```s
$ docker service update --force web
```

### Cleanup by removing the service

```s
$ docker service rm web
```

# Docker Health Checks

Does the container itself has a basic level of health?

Health checks look at the container command's code:
- `0`: OK
- `1`: Error

Container states:
0. `preparing`
1. `starting`
2. `healthy`
3. `unhealthy`

- Can see it healthcheck from `docker container ls`
- Can check last 5 with `docker container inspect`
- Docker does not respont to healthchecks
- Services will replace tasks if failing healthcheck


### Start a Postgres Database Server with No Healthcheck

```s
$ docker container run --name p1 -e POSTGRES_HOST_AUTH_METHOD=trust -d postgres
```

### Start Another with a Healthcheck

```s
$ docker container run --name p2 -e POSTGRES_HOST_AUTH_METHOD=trust -d --health-cmd="pg_isready -U postgres || exit 1" postgres
```

### Watch the container heath

```s
$ watch docker container ls
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS                            PORTS      NAMES
938401fcf586   postgres                 "docker-entrypoint.s…"   5 seconds ago    Up 3 seconds (health: starting)   5432/tcp   p2
e9a8344f3c29   postgres                 "docker-entrypoint.s…"   33 seconds ago   Up 28 seconds                     5432/tcp   p1
```

### Notice the health stats in the inspection

```s
$ docker container inspect p2
...
            "Health": {
                "Status": "healthy",
                "FailingStreak": 0,
                "Log": [
                    {
                        "Start": "2021-04-25T05:08:08.8487524+01:00",
                        "End": "2021-04-25T05:08:09.199273479+01:00",
                        "ExitCode": 0,
                        "Output": "/var/run/postgresql:5432 - accepting connections\n"
                    },
                    {
                        "Start": "2021-04-25T05:08:39.227757137+01:00",
                        "End": "2021-04-25T05:08:39.55723717+01:00",
                        "ExitCode": 0,
                        "Output": "/var/run/postgresql:5432 - accepting connections\n"
                    },
                    {
                        "Start": "2021-04-25T05:09:09.576211133+01:00",
                        "End": "2021-04-25T05:09:09.95579183+01:00",
                        "ExitCode": 0,
                        "Output": "/var/run/postgresql:5432 - accepting connections\n"
                    },
                    {
                        "Start": "2021-04-25T05:09:39.980050662+01:00",
                        "End": "2021-04-25T05:09:40.337518308+01:00",
                        "ExitCode": 0,
                        "Output": "/var/run/postgresql:5432 - accepting connections\n"
                    },
                    {
                        "Start": "2021-04-25T05:10:10.360389148+01:00",
                        "End": "2021-04-25T05:10:10.669898016+01:00",
                        "ExitCode": 0,
                        "Output": "/var/run/postgresql:5432 - accepting connections\n"
                    }
                ]
            }
...
```

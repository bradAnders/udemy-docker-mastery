# Basic Container Commands

Make sure proper install and client can talk to server

```docker
PS C:\Users\Brad> docker version

Client: Docker Engine - Community
 Cloud integration: 1.0.9
 Version:           20.10.5
...
Server: Docker Engine - Community
 Engine:
  Version:          20.10.5
...
```

Get more info
```docker
PS C:\Users\Brad> docker info
Client:
 Context:    default
...
Server:
 Containers: 1
...
```

See the available commands

```shell
PS C:\Users\Brad> docker

Usage:  docker [OPTIONS] COMMAND

A self-sufficient runtime for containers

Options:
      --config string      Location of client config files (default
...
  -v, --version            Print version information and quit

Management Commands:
  app*        Docker App (Docker Inc., v0.9.1-beta3)
...
  config      Manage Docker configs
  container   Manage containers
...
```

Deault Docker registry is `hub.docker.com`

### Run a container

```docker
PS C:\Users\Brad> docker container run --publish 80:80 nginx
```

- `docker container run ... nginx` downloads image `nginx` from Docker Hub into a local container and runs it
- `--publish` starts a new container from that image
- `80:80` routes traffic from device port 80 to the container port 80

Test it out at [`localhost`](http://localhost) in your browser

- `--detach` runs the container in the background, and returns the unique ID


```docker
PS C:\Users\Brad> docker container run --publish 80:80 --detach nginx
```

### List the running containers


```docker
PS C:\Users\Brad> docker container ls
CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS              PORTS                NAMES
91b648722055   nginx     "/docker-entrypoint.…"   About a minute ago   Up About a minute   0.0.0.0:80->80/tcp   happy_roentgen
```

### List all containers

```docker
PS C:\Users\Brad> docker container ls -a
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS                        PORTS                    NAMES
91b648722055   nginx          "/docker-entrypoint.…"   4 minutes ago   Up 4 minutes                  0.0.0.0:80->80/tcp       happy_roentgen
e33f84353035   nginx          "/docker-entrypoint.…"   9 minutes ago   Exited (0) 4 minutes ago                               clever_keldysh
8847030d34e9   560572ad3d3a   "waitress-serve --li…"   3 days ago      Exited (255) 34 minutes ago   0.0.0.0:8080->8080/tcp   stoic_brahmagupta
```

### Run with an environment variable

```docker
docker container run -e MYSQL_RANDOM_ROOT_PASSWORD=yes --name mysql mysql
```

### Start an existing container

```docker
docker container start mysql
```

### Check the reponse of the running container

`curl` calls an HTTP GET by default, but can be configured for other methods and protocols

```docker 
curl localhost
```

#### Note about cURL

Commands like `curl localhost:8080` do not work on Windows like they do on Unix because the Powershell `curl` is just an alias for `Invoke-WebRequest` (See aliases by calling `Get-Alias`). [Here is a guide](http://thesociablegeek.com/Azure/using-curl-in-powershell/) for installing the proper `curl` command. Test it out:

```docker 
curl localhost:8080
```

### Stop multiple docker containers

```docker
docker container stop mysql httpd nginx
```

### Remove multiple docker containers

```docker
docker container rm mysql httpd nginx 
```

### Check for the local images after container deletion

```shedockerll
docker image ls
```
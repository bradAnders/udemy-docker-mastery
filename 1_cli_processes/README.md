# CLI Process Monitoring

### List the processes running inside a single container

```docker
docker container top mysql
```

### Show the configuration of the how a single container was started

This shows nothing about the active container and what it's doing

```docker
docker container inspect mysql
```

### Use the `container stats` command instead to inspect multiple containers

```docker
docker container stats --help

Usage:  docker container stats [OPTIONS] [CONTAINER...]

Display a live stream of container(s) resource usage statistics

Options:
  -a, --all             Show all containers (default shows just running)
      --format string   Pretty-print images using a Go template
      --no-stream       Disable streaming stats and only pull the first result
      --no-trunc        Do not truncate output
```

Run this with no arguments to get a live view of the running containers

```docker
docker container stats
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
fa54b5335416   mysql     0.53%     352.6MiB / 12.34GiB   2.79%     936B / 0B     0B / 0B     38  
ac8d6ffff81b   nginx     0.00%     4.219MiB / 12.34GiB   0.03%     1.12kB / 0B   0B / 0B     2 
```

## CLI Container Interaction

### Start a new container interactively (No need for SSH)

```docker
docker container run -it
```

See `
docker container run --help` for information regarding arguments

`-t` Allocates a psuedo-TTY, similar to SSH

`-i`  Keeps STDIN open even if not attached


### Run additional commands in an existing container

```docker
docker container exec -it
```

### Run a new `nginx` container and get a `bash` prompt

```docker
> docker container run -it --name proxy nginx bash
root@c486cc6666e5:/# ls -al
total 80
drwxr-xr-x   1 root root 4096 Apr 10 20:00 .
drwxr-xr-x   1 root root 4096 Apr 10 20:00 ..
...
root@c486cc6666e5:/# exit
exit
> 
```

Because the command passed into the nginx container was just `bash`, and containers only run as long as the startup command runs, the container stopped when we stopped `bash`.

### Run a full Ubuntu image

```docker
> docker container run -it --name ubuntu ubuntu   
Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
a70d879fa598: Pull complete
...
Status: Downloaded newer image for ubuntu:latest
root@d05d91ad2509:/# apt-get update
Get:1 http://security.ubuntu.com/ubuntu focal-security InRelease [109 kB]
Get:2 http://archive.ubuntu.com/ubuntu focal InRelease [265 kB]
...
Reading package lists... Done
root@d05d91ad2509:/# apt-get install -y curl
Reading package lists... Done
...
done.
root@d05d91ad2509:/# curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
...
</BODY></HTML>
root@d05d91ad2509:/# exit
exit
>
```

This container can be stopped and restarted, and it will contain the same installations, etc.

```docker 
docker container start -ai ubuntu
root@d05d91ad2509:/# curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
...
</BODY></HTML>
root@d05d91ad2509:/# exit
exit
> 
```

### Use a shell inside a running container

```docker
docker container exec  --help

Usage:  docker container exec [OPTIONS] CONTAINER COMMAND [ARG...]       

Run a command in a running container

Options:
  -d, --detach               Detached mode: run command in the background
      --detach-keys string   Override the key sequence for detaching a   
                             container
  -e, --env list             Set environment variables
      --env-file list        Read in a file of environment variables     
  -i, --interactive          Keep STDIN open even if not attached        
      --privileged           Give extended privileges to the command     
  -t, --tty                  Allocate a pseudo-TTY
  -u, --user string          Username or UID (format:
                             <name|uid>[:<group|gid>])
  -w, --workdir string       Working directory inside the container      
```

### Use `bash` on the mysql container to install `procps` to use `ps` to view processes inside the container

Exiting `bash` will keep the container running because `docker container exec` calls an additional command, and is not the root process of the comainer.

```docker
> docker container ls
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                 NAMES
fa54b5335416   mysql     "docker-entrypoint.s…"   37 minutes ago   Up 37 minutes   3306/tcp, 33060/tcp   mysql
...
> docker container exec -it mysql bash
root@fa54b5335416:/# ps aux
bash: ps: command not found
root@fa54b5335416:/# apt-get update && apt-get install -y procps
Get:1 http://deb.debian.org/debian buster InRelease [121 kB]
...
Processing triggers for libc-bin (2.28-10) ...
root@fa54b5335416:/# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
mysql        1  0.5  2.8 2056888 370384 ?      Ssl  19:47   0:10 mysqld
mysql       90  0.2  0.0      0     0 ?        Z    19:47   0:04 [mysqld] <defunct>
root       192  0.0  0.0   3864  3248 pts/0    Ss   20:22   0:00 bash
root       597  0.0  0.0   7636  2724 pts/0    R+   20:23   0:00 ps aux
root@fa54b5335416:/# exit
exit
> docker container ls
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                 NAMES
fa54b5335416   mysql     "docker-entrypoint.s…"   37 minutes ago   Up 37 minutes   3306/tcp, 33060/tcp   mysql
...
```

### Some Linux distros are very small and do not even come with `bash` installed. Alpine, for instance, has `sh` installed on its base image.

`bash` can be installed through `apk`, but more than doubles the container size.

```docker
> docker pull alpine
Using default tag: latest
...
docker.io/library/alpine:latest
> docker image ls
REPOSITORY                                                                           TAG       IMAGE ID       CREATED       SIZE  
...
alpine                                                                               latest    49f356fa4513   10 days ago   5.61MB
...
> docker container run -it alpine bash
docker: Error response from daemon: OCI runtime create failed: container_linux.go:370: starting container process caused: exec: "bash": executable file not found in $PATH: unknown.
> docker container run -it alpine sh  
/ # apk add bash
fetch https://dl-cdn.alpinelinux.org/alpine/v3.13/main/x86_64/APKINDEX.tar.gz
...
OK: 8 MiB in 18 packages
/ # bash
bash-5.1# 
```

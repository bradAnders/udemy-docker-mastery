
# Container Lifetime and Persistent Data

- Containers are immutable and ephemeral
- Data Volumes
  - Creates a location outside of the container's Union File System
  - Container sees a local file path
- Bind Mounts
  - Link container path to a host path
  - Container sees a local file path

# Data Volumes

List volumes

```docker
> docker volume ls
DRIVER    VOLUME NAME
local     fb8fa9cb58c6f954718cef18e27a75cb5f02aa9c3bfac6df62c3c31369181ad3
```

Inspect a volume
```docker
> docker volume inspect fb8fa9cb58c6f954718cef18e27a75cb5f02aa9c3bfac6df62c3c31369181ad3
[
    {
        "CreatedAt": "2021-04-17T18:36:40Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/fb8fa9cb58c6f954718cef18e27a75cb5f02aa9c3bfac6df62c3c31369181ad3/_data",
        "Name": "fb8fa9cb58c6f954718cef18e27a75cb5f02aa9c3bfac6df62c3c31369181ad3",
        "Options": null,
        "Scope": "local"
    }
]
```

## Named Volumes

- Create a new volume, setting the path a container will see to  `/var/lib/mysql`
  - This is the same as omiitting the `-v /var/lib/mysql` argument since this is the default location

```docker
> docker run -d --name mysql2 -e MYSQL_RANDOM_ROOT_PASSWORD=yes -v /var/lib/mysql mysql
```

- Use the volume named `mysql-db`
  - If the volume doesn't exist, create it

```docker
> docker run -d --name mysql -e MYSQL_RANDOM_ROOT_PASSWORD=yes -v mysql-db:/var/lib/mysql mysql
> docker run -d --name mysql2 -e MYSQL_RANDOM_ROOT_PASSWORD=yes -v mysql-db:/var/lib/mysql mysql
```

Two containers running, but using the same database

```docker
> docker volume ls   
DRIVER    VOLUME NAME
local     mysql-db
```

Inspect a volume
```docker
> docker volume inspect mysql-db
[
    {
        "CreatedAt": "2021-04-17T18:56:42Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/mysql-db/_data",
        "Name": "mysql-db",
        "Options": null,
        "Scope": "local"
    }
]
```

## Manually Create Volume

Specify a specific driver or metadata for a volume

```docker
> docker volume create --help
```

# Bind Mounting

- Maps a host file or directory to a container file or directory
- Skips UFS
- Host files "overwrite files" in container
- Cannot setup with Dockerfile

```docker
> cd .\dockerfile-sample-2\
dockerfile-sample-2> docker container run -d --name nginx -p 80:80 -v ${pwd}:/usr/share/nginx/html nginx
dockerfile-sample-2> docker container run -d --name nginx2 -p 8080:80 -v ${pwd}:/usr/share/nginx/html nginx 
```

Open the servers at `http://localhost/` and `http://localhost:8080/`

```bash
> docker container exec -it nginx bash
$ cd /usr/share/nginx/html
```

Notice that the entire local directory is mirrored in the container

```bash
$ touch test.txt
$ echo "it is me you are looking for" > testme.txt
```

Navigate to `http://localhost/testme.txt` or `http://localhost:8080/testme.txt` to see the file

# Database Volume Assignment


- Create a `postgres` container, version `9.6.1`
- Use Docker Hub to learn `VOLUME` path and version needed to run it
- Check logs, stop container
- Create a new `postgres` container with same named volumne using `9.6.2`
- Check logs to validate

  - This specific version does not run on amd. To run on raspberry pi, the following command was substituted:

    `docker container run -d --name psql -e POSTGRES_HOST_AUTH_METHOD=trust -v psql:/var/lib/postgresql/data postgres`

```s
docker container run -d --name psql -v psql:/var/lib/postgresql/data postgres:9.6.1
docker container logs -f psql
docker container stop psql
docker container run -d --name psql2 -v psql:/var/lib/postgresql/data postgres:9.6.2
docker container ps -a
docker volume ls
docker container logs -f psql2
docker container stop psql2
```

# Bind Mounts Assignment

- Using a Jekyll "Static Site Generator" to start a local web server

```
docker run -p 80:4000 --name jek -v ${pwd}:/site bretfisher/jekyll-serve
```
# Container Images

## What is in an image

- Application binaries
- Metadata on how to run them
- No kernel
- No kernel modules
  - The host provides the kernel

## Docker Hub Registry Images

[Docker Hub](https://hub.docker.com/)

[Docker Image Sources](https://github.com/docker-library/official-images/tree/master/library)

- Images are not named, they are tagged
- One image can have multiple tags
- Identity of an image is checked through a hash (e.g. SHA256)
- Best practive for development is to use the `latest` tag
- Best practive for production is to use a specific version tag

## Image Layers

- Images are built designed using the union file system concept
  - Define layers of changes
- Every image starts with a blank layer `scratch`
  - Every set of changes from `scratch` is another layer
- Common layers are shared between images behind the scenes to save of storage space, etc.
- Modifications to file system within a container are "copy on write"
  - The modified file is saved as a new layer

### Show the layers for an image

`<missing>` is a misnomer. The layers are just not their own image, so they don't get their own ID.

```docker
> docker image history nginx:latest
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
7ce4f91ef623   10 days ago   /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B
<missing>      10 days ago   /bin/sh -c #(nop)  STOPSIGNAL SIGQUIT           0B
<missing>      10 days ago   /bin/sh -c #(nop)  EXPOSE 80                    0B
<missing>      10 days ago   /bin/sh -c #(nop)  ENTRYPOINT ["/docker-entr…   0B
<missing>      10 days ago   /bin/sh -c #(nop) COPY file:09a214a3e07c919a…   4.61kB
<missing>      10 days ago   /bin/sh -c #(nop) COPY file:0fd5fca330dcd6a7…   1.04kB
<missing>      10 days ago   /bin/sh -c #(nop) COPY file:0b866ff3fc1ef5b0…   1.96kB
<missing>      10 days ago   /bin/sh -c #(nop) COPY file:65504f71f5855ca0…   1.2kB
<missing>      10 days ago   /bin/sh -c set -x     && addgroup --system -…   63.8MB
<missing>      10 days ago   /bin/sh -c #(nop)  ENV PKG_RELEASE=1~buster     0B
<missing>      10 days ago   /bin/sh -c #(nop)  ENV NJS_VERSION=0.5.3        0B
<missing>      10 days ago   /bin/sh -c #(nop)  ENV NGINX_VERSION=1.19.9     0B
<missing>      10 days ago   /bin/sh -c #(nop)  LABEL maintainer=NGINX Do…   0B
<missing>      11 days ago   /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>      11 days ago   /bin/sh -c #(nop) ADD file:b797b4d60ad7954e9…   69.3MB
```

### Get the metadata for the image

```docker
> docker image inspect nginx       
...
            "ExposedPorts": {
                "80/tcp": {}
            },
...
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "NGINX_VERSION=1.19.9",
                "NJS_VERSION=0.5.3",
                "PKG_RELEASE=1~buster"
            ],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ",
                "CMD [\"nginx\" \"-g\" \"daemon off;\"]"
            ],
...

        "Architecture": "amd64",
        "Os": "linux",
 ...
```

## Image Tagging

- `REPOSITORY` is the name of the image source, on Docker Hub for example.
  - Official repositories have have no `/`
  - Repositories from authors other than Docker are labeled `author/repo` (`mysql/mysql-server` is Oracle's repository called `mysql-server`)
- `TAG` is an identifier for the version of that image
  - It's essentially a pointer to an `IMAGE ID`
  - Can be dynamic - `latest` will change where it is pointing over time
- `IMAGE ID` is unique for each version of an image

```docker
> docker image ls               
REPOSITORY           TAG       IMAGE ID       CREATED        SIZE  
nginx                latest    7ce4f91ef623   10 days ago    133MB 
mysql                latest    e646c6533b0b   10 days ago    546MB 
httpd                latest    d5995e280a0e   10 days ago    138MB 
ubuntu               14.04     13b66b487594   2 weeks ago    197MB 
ubuntu               18.04     3339fde08fc3   2 weeks ago    63.3MB
mysql/mysql-server   latest    d320e74763f4   2 months ago   406MB 
centos               7         8652b9f0cb4c   4 months ago   204MB 
```

### Add a tag to an existing image

```shell
> docker image tag nginx bradtanders/nginx
> docker image ls
REPOSITORY           TAG       IMAGE ID       CREATED        SIZE  
bradtanders/nginx    latest    7ce4f91ef623   10 days ago    133MB 
nginx                latest    7ce4f91ef623   10 days ago    133MB 
mysql                latest    e646c6533b0b   10 days ago    546MB 
...
```

### Login to a docker repository hub

```shell
> docker login
Login with your Docker ID to push and pull images from Docker Hub. If you dont have a Docker ID, head over to https://hub.docker.com to create one.
Username: bradtanders
Password: 
Login Succeeded
```

### View local credentials
```shell
> cd ~
> cat .docker/config.json
{
        "auths": {
                "526520550780.dkr.ecr.us-west-2.amazonaws.com": {},
                "https://index.docker.io/v1/": {}
        },
        "credsStore": "desktop",
        "stackOrchestrator": "swarm"
}
```

### Push the image to the remote

```docker
> docker image push bradtanders/nginx                      
Using default tag: latest
The push refers to repository [docker.io/bradtanders/nginx]
1914a564711c: Mounted from library/nginx
db765d5bf9f8: Mounted from library/nginx
903ae422d007: Mounted from library/nginx
66f88fdd699b: Mounted from library/nginx
2ba086d0a00c: Mounted from library/nginx
346fddbbb0ff: Mounted from library/mysql
latest: digest: sha256:c137f6c852bfdf74694fe20693bb11e61b51e0b8c50d17dff881f2db05e65de9 size: 1570
PS C:\repos\udemy_docker_mastery> 
```

### Add a new tag and push to the remote

Notice that duplicate layers are automatically identified and not uploaded

```docker
> docker image tag nginx bradtanders/nginx:testing
> docker image push bradtanders/nginx:testing
The push refers to repository [docker.io/bradtanders/nginx]
1914a564711c: Layer already exists
db765d5bf9f8: Layer already exists
903ae422d007: Layer already exists
66f88fdd699b: Layer already exists
2ba086d0a00c: Layer already exists
346fddbbb0ff: Layer already exists
testing: digest: sha256:c137f6c852bfdf74694fe20693bb11e61b51e0b8c50d17dff881f2db05e65de9 size: 1570
```

## Creating Images (Dockerfile)

- Each stanza in a Dockerfile define a new layer (Order matters)

### Common layer commands

- `FROM image:tag` [REQUIRED] This is the first stanza (excluding comments) of a Dockerfile, and specifies the base image to start with
- `ENV FOO BAR` Sets the environment variable key `FOO` to the value `BAR`
-  `RUN command arg --kwarg val` Runs the command `command` with the args `arg --kward val` at build time
   -  Best practice is to group multiple commands with `&&` to keep setup commands to a single layer
   -  Docker handles log files if the container outputs to `stdout` and `stderr` ([Python stdio](https://pythonguides.com/python-stderr-stdin-and-stdout/))
- `EXPOSE 80 443` Exposes the ports `80` and `443` on the image since all ports are blocked by default
- `CMD ["nginx", "-g", "daemon off;"]` [REQUIRED] This is the last stanza (excluding comments) of a Dockerfile and represents the default command to run when the container starts up. It is not called at build time.

### Example Dockerfile and build command

```docker
FROM debian:stretch-slim

ENV NGINX_VERSION 1.13.6-1~stretch
ENV NJS_VERSION   1.13.6.0.1.14-1~stretch

RUN apt-get update \
	&& apt-get install --no-install-recommends --no-install-suggests -y gnupg1 \
	&& \
	NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \
	found=''; \
	for server in \
		ha.pool.sks-keyservers.net \
		hkp://keyserver.ubuntu.com:80 \
		hkp://p80.pool.sks-keyservers.net:80 \
		pgp.mit.edu \
	; do \
		echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
		apt-key adv --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
	done; \
	test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
	apt-get remove --purge -y gnupg1 && apt-get -y --purge autoremove && rm -rf /var/lib/apt/lists/* \
	&& echo "deb http://nginx.org/packages/mainline/debian/ stretch nginx" >> /etc/apt/sources.list \
	&& apt-get update \
	&& apt-get install --no-install-recommends --no-install-suggests -y \
						nginx=${NGINX_VERSION} \
						nginx-module-xslt=${NGINX_VERSION} \
						nginx-module-geoip=${NGINX_VERSION} \
						nginx-module-image-filter=${NGINX_VERSION} \
						nginx-module-njs=${NJS_VERSION} \
						gettext-base \
	&& rm -rf /var/lib/apt/lists/*

RUN ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

### Build from the Dockerfile in the `.` directory, using `-t` to give it the name `customnginx` 
  - Can optionally give it a tag as well, such as `customnginx:latest` (The default tag is `latest`)
  - This image will only be used locally, but if pushing to a repo, make sure to specify the author, such as `bradtanders/customnginx:latest`

```docker
> docker image build -t customnginx .
[+] Building 102.8s (5/6)
...
```

### Copy a file from the local directory to the image

- Use the `WORKDIR` stanza to change directories, *not* `cd`
  - `/usr/share/nginx/html` is the default assets directory for this image
  - We are overriding the default html file
- Because we are inheriting from an existing `nginx` image, we don't need to re-implement the `CMD` stanza

```docker
FROM nginx:latest

WORKDIR /usr/share/nginx/html

COPY index.html index.html
```

```shell
> docker image build -t nginx-with-html .
...
> docker container run -p 80:80 --rm nginx-with-html
...
```

- Browse to [localhost](http://localhost/) to test it out!

### Tag this image with the destination repository name

```docker
> docker image tag nginx-with-html:latest bradtanders/nginx-with-html:latest
```

## Image Cleanup

Inspect system usage

```docker
> docker system df
```

Remove "dangling" images

```docker
> docker image prune
```

Remove images with no associated container running

```docker
> docker image prune -a
```

Removes stopped containers, unused networks, dangling images, dangling build cache

```docker
> docker system prune
```


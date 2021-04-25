# Create a Private Registry

### Pull the offical image

```s
$ docker container run -d -p 5000:5000 --name registry registry
```

### Pull a small test image

```s
$ docker pull hello-world
$ docker run hello-world
```

### Push it to the local registry

```s
$ docker tag hello-world 127.0.0.1:5000/hello-world
$ docker image ls
REPOSITORY                   TAG       IMAGE ID       CREATED             SIZE
127.0.0.1:5000/hello-world   latest    851163c78e4a   15 months ago       4.85kB
hello-world                  latest    851163c78e4a   15 months ago       4.85kB
$ docker push 127.0.0.1:5000/hello-world
```


### Remove the local images

```s
$ docker image remove hello-world
$ docker container rm silly_mcnulty
$ docker image remove 127.0.0.1:5000/hello-world
REPOSITORY               TAG       IMAGE ID       CREATED             SIZE
```

### Pull from the new registry

```s
$ docker pull 127.0.0.1:5000/hello-world
$ docker image ls
REPOSITORY                   TAG       IMAGE ID       CREATED             SIZE
127.0.0.1:5000/hello-world   latest    851163c78e4a   15 months ago       4.85kB
```

### Set up the registry with a volume

```s
$ docker container kill registry 
$ docker container rm registry
$ docker container run --name registry -d -p 5000:5000 -v $(pwd)/registry-data:/var/lib/registry registry
$ docker push 127.0.0.1:5000/hello-world
$ ls -a registry-data
```

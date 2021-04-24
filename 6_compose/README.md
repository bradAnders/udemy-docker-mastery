
# Docker Compose

- YAML file for configuration
- `docker-compose` command to build

# YAML Structure

```yaml
version: '3.1'  # if no version is specified then v1 is assumed. Recommend v2 minimum

services:  # containers. same as docker run
  servicename: # a friendly name. this is also DNS name inside network
    image: # Optional if you use build:
    command: # Optional, replace the default CMD specified by the image
    environment: # Optional, same as -e in docker run
    volumes: # Optional, same as -v in docker run
  servicename2:

volumes: # Optional, same as docker volume create

networks: # Optional, same as docker network create
```

# Commands

### Run the docker-compose.yaml file

```s
docker-compose up
```

### Run a specific docker-compose.test.yaml file

```s
docker-compose -f docker-compose.test.yaml up
```

### Stop all services in docker-compose.yaml

```s
docker-compose down
```

### -- AND REMOVE VOLUMES

```s
docker-compose down -v
```

### -- And remove images built with docker-compose.yaml

```s
docker-compose down --rmi local
```
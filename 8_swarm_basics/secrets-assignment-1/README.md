# Secrets Assignment 1

### Compose file

```s
$ cd ~/udemy-docker-mastery/8_swarm_basics/secrets-assignment-1/
$ vi docker-compose.yml
```


### Commands

```s
$ scp docker-compose.yml node1:docker-compose.yml
$ ssh node1
$ echo "my password" | docker secret create psql-pw -
$ docker stack deploy -c docker-compose.yml drupal
$ docker service logs drupal_postgres 
```

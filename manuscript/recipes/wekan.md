---
description: A self-hosted Trello
---

# Wekan

Wekan is an open-source kanban board which allows a card-based task and to-do management, similar to tools like WorkFlowy or Trello.

![Wekan Screenshot](../images/wekan.jpg)

Wekan allows to create Boards, on which Cards can be moved around between a number of Columns. Boards can have many members, allowing for easy collaboration, just add everyone that should be able to work with you on the board to it, and you are good to go! You can assign colored Labels to cards to facilitate grouping and filtering, additionally you can add members to a card, for example to assign a task to someone.

There's a [video](https://www.youtube.com/watch?v=N3iMLwCNOro) of the developer showing off the app, as well as a [functional demo](https://boards.wekan.team/b/D2SzJKZDS4Z48yeQH/wekan-open-source-kanban-board-with-mit-license).

!!! note
    For added privacy, this design secures wekan behind a [traefik-forward-auth](/ha-docker-swarm/traefik-forward-auth/), so that in order to gain access to the wekan UI at all, authentication must have already occurred.

--8<-- "recipe-standard-ingredients.md"

## Preparation

### Setup data locations

We'll need several directories to bind-mount into our container, so create them in /var/data/wekan:

```bash
mkdir /var/data/wekan
cd /var/data/wekan
mkdir -p {wekan-db,wekan-db-dump}
```

### Prepare environment

Create `/var/data/config/wekan.env`, and populate with the following variables:

```yaml
MONGO_URL=mongodb://wekandb:27017/wekan
ROOT_URL=https://wekan.example.com
MAIL_URL=smtp://wekan@wekan.example.com:password@mail.example.com:587/
MAIL_FROM="Wekan <wekan@wekan.example.com>"

# Mongodb specific database dump details
BACKUP_NUM_KEEP=7
BACKUP_FREQUENCY=1d
```

### Setup Docker Swarm

Create a docker swarm config file in docker-compose syntax (v3), something like this:

--8<-- "premix-cta.md"

```yaml
version: '3'

services:
  wekandb:
    image: mongo:latest
    command: mongod --smallfiles --oplogSize 128
    networks:
      - internal
    volumes:
      - /var/data/runtime/wekan/database:/data/db
      - /var/data/wekan/database-dump:/dump

  wekan:
    image: wekanteam/wekan:latest
    networks:
      - internal
      - traefik_public
    env_file: /var/data/config/wekan/wekan.env
    deploy:
      labels:
        # traefik common
        - traefik.enable=true
        - traefik.docker.network=traefik_public

        # traefikv1
        - traefik.frontend.rule=Host:wekan.example.com
        - traefik.port=4180     

        # traefikv2
        - "traefik.http.routers.wekan.rule=Host(`wekan.example.com`)"
        - "traefik.http.services.wekan.loadbalancer.server.port=4180"
        - "traefik.enable=true"

        # Remove if you wish to access the URL directly
        - "traefik.http.routers.wekan.middlewares=forward-auth@file"

  db-backup:
    image: mongo:latest
    env_file : /var/data/config/wekan/wekan.env
    volumes:
      - /var/data/wekan/database-dump:/dump
      - /etc/localtime:/etc/localtime:ro
    entrypoint: |
      bash -c 'bash -s <<EOF
      trap "break;exit" SIGHUP SIGINT SIGTERM
      sleep 2m
      while /bin/true; do
        mongodump -h db --gzip --archive=/dump/dump_\`date +%d-%m-%Y"_"%H_%M_%S\`.mongo.gz
        (ls -t /dump/dump*.mongo.gz|head -n $$BACKUP_NUM_KEEP;ls /dump/dump*.mongo.gz)|sort|uniq -u|xargs rm -- {}
        sleep $$BACKUP_FREQUENCY
      done
      EOF'
    networks:
    - internal    

networks:
  traefik_public:
    external: true
  internal:
    driver: overlay
    ipam:
      config:
        - subnet: 172.16.3.0/24
```

--8<-- "reference-networks.md"

## Serving

### Launch Wekan stack

Launch the Wekan stack by running ```docker stack deploy wekan -c <path -to-docker-compose.yml>```

Log into your new instance at `https://**YOUR-FQDN**`, with user "root" and the password you specified in `wekan.env`.

[^1]: If you wanted to expose the Wekan UI directly, you could remove the traefik-forward-auth from the design.

--8<-- "recipe-footer.md"

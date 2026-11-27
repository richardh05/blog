---
title: "How to auto-backup a home-server"
pubDatetime: 2026-11-27T18:30:00
featured: false
draft: false
tags:
  - self-hosting
# ogImage: ../../assets/images/example.png # src/assets/images/example.png
description: Covers PostgreSQL, SQLite and media volumes.
---

Seemingly every self-hosting guide stresses the importance of backups, but leaves out how to actually do it. In fact, it doesn't even seem like there is a community-wide solution for backups in the same way that everyone has rallied behind services like Immich for photos and Plex/Jellyfin for entertainment. It's not always as simple as just copy-pasting the volumes your containers have mounted; SQLite can very easily corrupt if copied while in use, and PostgreSQL's raw data isn't really portable between machines. Some services have their own integrated backup features, but it's hit or miss. I'd like to share what I've found to be the most low-friction way of backing up all data on my home server, including active PostgreSQL and SQLite databases. By the end, you'll have:

- Portable backups for PostgreSQL, SQLite, and other cold data mounted by your Docker services.
- One web dashboard for all backup routines.
- 3-2-1 compliance, storing the data locally and/or on cloud storage.
- Guaranteed safety from database corruption.
- Full coverage of *every* container in your stack.

We'll achieve this by hosting a backup solution named Backrest, which offers a single web GUI for all your backups. For this guide, I'm assuming that:
- You're using Docker Compose for your home server. I use Docker, but Podman should also be fine.
- You have a variable named `${DATA}` which points to a directory where *all* of your containers store their persistent data (e.g., `${DATA}/jellyfin:/config` for Jellyfin).

## Prerequisite: Move everything to PostgreSQL
If a service supports PostgreSQL, and you have it set up for SQLite or MariaDB instead, I'd strongly recommend you migrate it to PostgreSQL. Specifically, you want everything to be connected to the *same* PostgreSQL container with a different database, username, and password for each service. This will save a *lot* of system resources compared to having one PostgreSQL container for each service, and this guide assumes you've used this approach later on. 

Having a centralised database server rather than fragmented SQLite files means they can be backed up easily without being shut down. While MariaDB *could* also work for this use case, plenty of popular services like Immich, Miniflux, and Linkwarden *only* support PostgreSQL, making it easier to standardise on if you run one of them.

In my example, I'm using the `tensorchord/vchord-postgres` image rather than the standard `postgres` image. It contains the `pgvector` and `vectorchord` extensions my Immich instance needs; make sure your database has all the extensions it needs too.

```yml
services:
  postgres-shared:
    image: tensorchord/vchord-postgres:pg18-v1.1.1
    container_name: postgres-shared
    restart: unless-stopped
    command: ["postgres", "-c", "shared_preload_libraries=vchord.so"]
    networks:
      - postgres-shared-network
    environment:
      POSTGRES_USER: postgres_user
      POSTGRES_PASSWORD: ${SHARED_PG_PASSWORD}
    volumes:
      - ${DATA}/postgres-shared:/var/lib/postgresql/
      - ${PG_BACKUP_DUMP}:/backups
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres_user"]      
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
```

## Labelling SQLite-based services
For those services that only support SQLite, like Audiobookshelf, we'll need a way for our backup pipeline to know that it's using SQLite. The simplest way is to give it a label in the `labels` block, explicitly marking it to be shut down before a backup to avoid database corruption. I use the label `docker-volume-backup.stop-during-backup: "true"`, which is a common convention, though you could change this if you wanted.

```yml
services: 
  audiobookshelf:
    image: ghcr.io/advplyr/audiobookshelf:latest
    container_name: audiobookshelf
    ports:
      - 80:80
    volumes:
      - ${AUDIOBOOKS}:/audiobooks
      - ${DATA}/audiobookshelf/config:/config
      - ${DATA}/audiobookshelf/metadata:/metadata
    labels:
      docker-volume-backup.stop-during-backup: "true"
      # Alternative Syntax:
      # - docker-volume-backup.stop-during-backup=true
```

## Configure Backrest
Backrest is the most set-it-and-forget-it solution I've found. It's a web-based interface for restic, offering scheduling, repository management, and secure file restoration. Restic, the tool it uses under the hood, is capable of deduplicating data across different timed snapshots. If your first PostgreSQL snapshot is 160MB, and your second is 163MB with 5MB changed, the backup repository should only use 165MB total for the two. Restic also compresses data wherever possible—my PostgreSQL dump only took up around 80MB, down from 160MB when I first ran it. I won't be giving a full tour of the Backrest UI, so check their [getting started](https://garethgeorge.github.io/backrest/introduction/getting-started) page if you want to understand the workflow a little better.

For database backups, it's crucial that your Backrest image has access to the `docker-cli` tool. The official Backrest image is built on Alpine Linux, where the tool can be added with `apk add --no-cache docker-cli`. Backrest will also need access to your host's docker socket, either by giving it direct access or by using a socket proxy with scoped privileges as I have in my example. This grants Backrest a lot of control over your system, so take care not to expose it to the public.

```yml
networks:
  backrest-socket:
    internal: true

volumes:
  backrest_cache:
  backrest_tmp:

services:
  backrest:
    build:
      context: .
      dockerfile_inline: |
        FROM ghcr.io/garethgeorge/backrest:latest
        RUN apk add --no-cache docker-cli
    container_name: backrest
    depends_on:
      - socket-proxy
    volumes:
      - ${DATA}/backrest:/data
      - ${DATA}/backrest-config:/config
      - backrest_cache:/cache
      - backrest_tmp:/tmp
      # Target local repository storage
      - ${BACKUPS}:/backups
      # Source data volumes
      - ${PG_BACKUP_DUMP}:/pg-dumps:ro
      - ${DATA}:/volumes:ro
    environment:
      - TZ=Europe/London
      - DOCKER_HOST=tcp://socket-proxy:2375
    networks:
      - backrest-socket
    ports:
      - 9898:9898
  
  socket-proxy:
    image: tecnativa/docker-socket-proxy:latest
    container_name: backrest-socket-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - CONTAINERS=1
      - ALLOW_STOP=1
      - ALLOW_START=1
      - EXEC=1
    networks:
      - backrest-socket
```

## Set up the PostgreSQL plan
Launch the container and [create your first repository](https://garethgeorge.github.io/backrest/introduction/getting-started#_2-repository-setup). It's up to you whether you want to use a local directory, an S3 bucket or any other storage type Backrest supports. Once you're set up, click "Add Plan". Set the name to something like "PostgreSQL", select the repo you just made, and set one "Paths" entry to `/pg-dumps`.

Scroll down to "Hooks" and add a `CONDITION_SNAPSHOT_START`, which will run before your backup does. Add the following in the "Script" textbox.

```sh
docker exec postgres-shared pg_dumpall -U postgres_user --clean --if-exists -f /backups/postgres_all.sql
```
Check that the command works in the terminal; you may need to adjust the container name, username, or file path if yours are different. Press "Submit", and you should now have a backup routine for your PostgreSQL instance that grabs the latest database dump before archiving it.

## Set up the Docker Volumes plan

Add another plan called something like "Docker-Volumes". The path, for my example, should be `/volumes`. The one downside to this approach of backing up near-enough every container's data is that a lot of them will mix large cache files in with their persistent data. Unless you want to throw more storage or money at this problem, you'll probably want to take some time to look at each of your containers' data directories and write some entries in the "Excludes" section to make sure they're ignored by Backrest. Here's an example of my ignore list from Backrest's config.json (if pasting into the Web UI, omit the quotes and commas). You'll definitely want to ignore PostgreSQL (because we just used a custom approach), Kavita's cache (which stores decompressed manga/comics), and some Jellyfin paths:

```js
"excludes": [
  "/volumes/*postgres*",
  "/volumes/*redis*",
  "/volumes/*mariadb*",
  "/volumes/*tailscale*",
  "/volumes/*romm*",
  "**/jellyfin/**/metadata",
  "**/jellyfin/**/cache",
  "**/jellyfin/**/transcodes",
  "**/jellyfin/**/trickplay",
  "**/jellyfin/**/data/splashscreen.png",
  "**/paperless/**/*log*",
  "**/paperless/**/classification_model.pickle",
  "**/qbittorrent/**/data",
  "**/qbittorrent/**/downloads",
  "**/syncthing/**/index-*",
  "**/syncthing/**/*.log",
  "**/mealie/**/backups",
  "**/mealie/**/mealie.log",
  "**/kavita/**/backups",
  "**/kavita/**/covers",
  "**/kavita/**/temp",
  "**/kavita/**/images",
  "**/kavita/**/cache.db",
  "**/job-ops/visa-sponsors",
  "**/dockhand/icon-cache",
  "**/backrest",
  "**/cache",
  "**/cache-long"
],
```
In the "Hooks" section, add a `CONDITION_SNAPSHOT_START` hook. This should stop all containers with the `docker-volume-backup.stop-during-backup=true` label, and keep a record of what containers have been stopped in the `/tmp` directory to bring them back online later.

```sh
docker ps -q --filter "label=docker-volume-backup.stop-during-backup=true" > /tmp/stopped_containers.txt && \
if [ -s /tmp/stopped_containers.txt ]; then
  docker stop $(cat /tmp/stopped_containers.txt)
fi
```

Next, add a `CONDITION_SNAPSHOT_END` hook to restart the containers from that temporary log we kept.
```sh
if [ -s /tmp/stopped_containers.txt ]; then
  docker start $(cat /tmp/stopped_containers.txt) && rm -f /tmp/stopped_containers.txt
fi
```
Press "Submit", and Backrest should now stop all SQLite-based containers, back up their data (along with cold data used by PostgreSQL services with volume mounts), and bring the containers back online.
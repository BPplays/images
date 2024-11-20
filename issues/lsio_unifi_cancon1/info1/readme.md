i can't connect to unifi-network-application over port 8443
unifi-network-application can be pinged by another container on main but chrome says This site can’t be reached
```
root@2b85b05af3fd:/# ping6 unifi-network-application
PING unifi-network-application (fd0e:a882:618c:ff01::f): 56 data bytes
64 bytes from fd0e:a882:618c:ff01::f: seq=0 ttl=64 time=0.215 ms
64 bytes from fd0e:a882:618c:ff01::f: seq=1 ttl=64 time=0.047 ms
64 bytes from fd0e:a882:618c:ff01::f: seq=2 ttl=64 time=0.053 ms
^C
--- unifi-network-application ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.047/0.105/0.215 ms
```
```
root@2b85b05af3fd:/# ping unifi-network-application
PING unifi-network-application (172.29.94.15): 56 data bytes
64 bytes from 172.29.94.15: seq=0 ttl=64 time=0.039 ms
64 bytes from 172.29.94.15: seq=1 ttl=64 time=0.052 ms
^C
--- unifi-network-application ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.039/0.045/0.052 ms
```

my logs
```
> docker logs unifi-network-application
[migrations] started
[migrations] no migrations found
───────────────────────────────────────

      ██╗     ███████╗██╗ ██████╗
      ██║     ██╔════╝██║██╔═══██╗
      ██║     ███████╗██║██║   ██║
      ██║     ╚════██║██║██║   ██║
      ███████╗███████║██║╚██████╔╝
      ╚══════╝╚══════╝╚═╝ ╚═════╝

   Brought to you by linuxserver.io
───────────────────────────────────────

To support LSIO projects visit:
https://www.linuxserver.io/donate/

───────────────────────────────────────
GID/UID
───────────────────────────────────────

User UID:    0
User GID:    1000
───────────────────────────────────────
Linuxserver.io version: 8.6.9-ls69
Build-date: 2024-11-12T17:34:23+00:00
───────────────────────────────────────

[custom-init] No custom files found, skipping...
```

iptables:
```
> sudo ip6tables -L -n | grep 8443
ACCEPT     6    --  ::/0                 fd0e:a882:618c:ff01::f  tcp dpt:8443
> sudo iptables -L -n | grep 8443
ACCEPT     6    --  0.0.0.0/0            172.29.94.15         tcp dpt:8443
```


here is my docker-compose
```yaml
name: unifi-network-application
services:
  unifi-db:
    container_name: unifi-db
    image: docker.io/mongo:7
    restart: always

    configs:
      - source: init-mongo.js
        target: /docker-entrypoint-initdb.d/init-mongo.js
    environment:
      - PGID=1000
      - PUID=1000
      - "TZ=${GLOBAL_TZ}"
    ports:
      - 27017
    volumes:
      - type: bind
        source: ${DOCKER_VOLUMES_NETWORK_FOLDER_WORKAROUND}/unifi-network-application/data
        target: /data/db
        read_only: false
        bind:
          propagation: rslave
    networks:
     - unifi-bridge
    privileged: false

  unifi-db-backup:
    container_name: unifi-db-backup
    image: tiredofit/db-backup
    networks:
     - unifi-bridge
    volumes:

      - type: bind
        source: ${DOCKER_VOLUMES_NETWORK_FOLDER_WORKAROUND}/unifi-network-application/db_backups
        target: /backup
        read_only: false
        bind:
          propagation: rslave
    environment:
      - TIMEZONE=${GLOBAL_TZ}
      - CONTAINER_NAME=unifi-db
      - CONTAINER_ENABLE_MONITORING=FALSE

      - BACKUP_JOB_CONCURRENCY=${DB_BACKUP_DEF_JOB_CONCURRENCY}     # Only run one job at a time
      - DEFAULT_CHECKSUM=${DB_BACKUP_DEF_CHECKSUM}        # Don't create checksums
      - DEFAULT_COMPRESSION=${DB_BACKUP_DEF_COMPRESSION}     # Compress all with ZSTD
      - DEFAULT_BACKUP_INTERVAL=${DB_BACKUP_DEF_INTERVAL}   # Backup every 1440 minutes
      - DEFAULT_BACKUP_BEGIN=${DB_BACKUP_DEF_BEGIN}      # Start backing up at midnight
      - DEFAULT_CLEANUP_TIME=${DB_BACKUP_DEF_CLEANUP_TIME}    # Cleanup backups after a week

      - DB01_TYPE=mongo
      - DB01_HOST=unifi-db
      - DB01_NAME=unifi-db
      - DB01_USER=unifi
      - DB01_PASS=censored_mango_pass
    restart: always


  unifi-network-application:
    container_name: unifi-network-application
    depends_on:
      unifi-db:
        condition: service_started
        required: true
    environment:
      - MONGO_DBNAME=unifi-db
      - MONGO_HOST=unifi-db
      - MONGO_PASS=censored_mango_pass
      - MONGO_PORT=27017
      - MONGO_USER=unifi
      - MONGO_AUTHSOURCE=admin
      - MEM_LIMIT=1024 #optional
      - MEM_STARTUP=1024 #optional
      # - PUID=1000
      - PGID=1000
      # - PGID=0
      - PUID=0
      - TZ=${GLOBAL_TZ}
    image: lscr.io/linuxserver/unifi-network-application:latest
    ports:
      - 8443:8443
      - 3478:3478/udp
      - 10001:10001/udp
      - 8080:8080
      - 1900/udp #optional
      # - 1900:1900/udp #optional
      - 8843:8843 #optional
      - 8880:8880 #optional
      - 6789:6789 #optional
      - 5514:5514/udp #optional
    restart: always
    volumes:
      - type: bind
        source: ${DOCKER_VOLUMES_NETWORK_FOLDER}/unifi-network-application/config
        target: /config
        read_only: false
        bind:
          propagation: rslave
    networks:
      - main
      - unifi-bridge


networks:
  main:
    name: main
    external: true
    driver: bridge


  unifi-bridge:
    driver: bridge
    enable_ipv6: true
  default:
    driver: bridge
    enable_ipv6: true

configs:
  init-mongo.js:
    content: |
      db.getSiblingDB("unifi-db").createUser({user: "unifi", pwd: "censored_mango_pass", roles: [{role: "dbOwner", db: "unifi-db"}]});
      db.getSiblingDB("unifi-db_stat").createUser({user: "unifi", pwd: "pass", roles: [{role: "dbOwner", db: "unifi-db_stat"}]});
```
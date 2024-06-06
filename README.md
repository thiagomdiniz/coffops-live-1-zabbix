# #1 Live Coffops - Zabbix 6.4 + TimescaleDB

This repository contains the code I used in a live on the Coffops YouTube Channel:
- https://www.youtube.com/watch?v=3-UOVAbMyU4

## Doc reference

- Zabbix
    - https://www.zabbix.com/container_images
    - https://www.zabbix.com/documentation/6.4/en/manual/installation/containers
    - https://git.zabbix.com/projects/ZBX/repos/zabbix/browse?at=refs%2Ftags%2F6.4.15
- Docker Compose
    - https://docs.docker.com/compose/compose-file/
- TimescaleDB (postgresql extension to make SQL scalable for time-series data)
    - https://docs.timescale.com/self-hosted/latest/install/installation-docker/
    - https://github.com/timescale/timescaledb-docker
    - https://hub.docker.com/r/timescale/timescaledb
    - https://docs.timescale.com/use-timescale/latest/hypertables/about-hypertables/

## Set DB secrets

- `cd zabbix-docker`
- Put DB user in file `./env_vars/.POSTGRES_USER`
- Puts DB password in file `./env_vars/.POSTGRES_PASSWORD`
- Set files permissions:
    ```
    sudo chown 70:1995 env_vars/.POSTGRES_*
    sudo chmod 440 env_vars/.POSTGRES_*
    ```

## Start containers

The `--profile dev` parameter include the `postgres-server` and `mail-server` services for dev/testing purposes.

```
docker compose --profile dev up -d
docker compose --profile dev ps -a
```

Useful command to reload Zabbix configurations:

```
docker compose exec zabbix-server zabbix_server -R config_cache_reload
```

## Check postgres/timescaledb

```
docker compose logs postgres-server |less
docker compose exec postgres-server psql -U zabbix -d zabbix
\du #list roles
\l  #list databases
\dx #list installed extensions
\dt #list tables
select * from _timescaledb_catalog.hypertable; #show hypertables
```

## Check zabbix services

```
docker compose logs zabbix-server |less
docker compose logs zabbix-web |less
docker compose logs zabbix-web-service |less
```

Access the Zabbix frontend:
- http://localhost:8080/

## Check timezone

```
docker compose exec mail-server date
docker compose exec postgres-server date
docker compose exec zabbix-server date
docker compose exec zabbix-web date
docker compose exec zabbix-web-service date
```
- Proceed to the Administration -> General -> GUI frontend menu section
- Check latest data datetime

## Check housekeeping conf

- Proceed to the Administration -> Housekeeping frontend menu section.

## Finish zabbix-web-service configuration

A Frontend URL parameter should be set to enable communication between Zabbix frontend and Zabbix web service:
- Proceed to the Administration -> General -> Other parameters frontend menu section
- Specify the full URL of the Zabbix web interface in the Frontend URL parameter
  - http://zabbix-web:8080/ (use service name from docker compose)
  - docker compose --profile dev logs zabbix-web-service -f

## Set Email media type and test

- Proceed to the Alerts -> Media types frontend menu section and edit/enable Email (use the `mail-server` service, smtp port `1025`)
- Add the media type to the zabbix user
- Reports -> Scheduled reports -> add and test
- Show containers logs and zabbix processes graphs
- To read the email sent by zabbix: http://localhost:1080/

## Show rebranding

- Enabled zabbix-web rebranding volume:
    - `- ${REBRANDING_DIRECTORY}/:/usr/share/zabbix/local/conf/:ro`
- Apply changes: `docker compose --profile dev up -d zabbix-web`

## Docker monitoring

- Install `zabbix-agent2`
    - https://www.zabbix.com/download?zabbix=6.4&os_distribution=red_hat_enterprise_linux&os_version=9&components=agent_2&db=&ws=
    - Adjust zabbix-agent2 configuration:
        ```
        LogFileSize=200
        Server=172.16.238.4
        ServerActive=172.16.238.4
        Plugins.SystemRun.LogRemoteCommands=1
        ```
    - `sudo systemctl enable --now zabbix-agent2.service`
- https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/app/docker?at=refs%2Ftags%2F6.4.15
- Get docker host IP and configure it in Zabbix host iface
- Add Docker template to the zabbix host
- Show item error: permission denied
- Fix error: Add zabbix user to docker group
    - `sudo usermod -a -G docker zabbix`
    - `sudo systemctl restart zabbix-agent2.service`
- Add dashboard graphs to Docker template (graph prototype -> cpu,mem,net)


## Database monitoring

- https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/db/postgresql_odbc?at=refs%2Ftags%2F6.4.15
- Add `PostgreSQL by ODBC` template to Zabbix host
- Create DB user:
    ```
    CREATE USER zbx_monitor WITH PASSWORD 'zabbix' INHERIT;
    GRANT pg_monitor TO zbx_monitor;
    ```
- Define host macros
- Show ODBC error on a host item
- Create custom `zabbix-server` docker image:
    ```
    build:
        context: ./Dockerfiles/zabbix-server
        args:
            ZABBIX_SERVER_IMAGE: ${ZABBIX_SERVER_IMAGE}
            ZABBIX_IMAGE_TAG: ${ZABBIX_IMAGE_TAG}
            ZABBIX_IMAGE_TAG_POSTFIX: ${ZABBIX_IMAGE_TAG_POSTFIX}
    ```
- Rebuild container `docker compose --profile dev up -d zabbix-server`
- Change Driver path to `/usr/lib/x86_64-linux-gnu/odbc/psqlodbcw.so`
- Execute discovery rules
- Show dashboard
- In production, use a `zabbix-proxy` for ODBC monitoring!

## Show trigger example

Show problem/dashboard and increase the `ZBX_CACHESIZE` value in the `./env_vars/.env_srv` file.

- https://www.zabbix.com/documentation/6.4/en/manual/appendix/config/zabbix_server
- https://www.zabbix.com/documentation/6.4/en/manual/concepts/server#server-process-types

To apply the new conf:
```
docker compose --profile dev up -d zabbix-server
```

## Stop containers and clean volumes

```
docker compose --profile dev down -v
sudo rm -rf zbx_env/
```

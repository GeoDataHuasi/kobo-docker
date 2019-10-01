# kobo-docker

1. [Introduction](#introduction)
1. [Important notice when upgrading from commit `5c2ef02` (March 4, 2019) or earlier](#important-notice-when-upgrading-from-commit-5c2ef02-march-4-2019-or-earlier)
2. [Important notice when upgrading from `[TODO: INSERT FINAL 1DB RELEASE HERE]` or earlier](#important-notice-when-upgrading-from-todo-insert-final-1db-release-here-or-earlier)
3. [Architecture](#architecture)
4. [Setup procedure](#setup-procedure)
5. [Usage](#usage)
    - Start/stop
    - Backups
    - Restore backups
    - Maintenance
7. [Warning](#warning)
8. [Troubleshooting](#troubleshooting)
9. [Redis performance](#redis-performance)


## Introduction 

`kobo-docker` is used to run a copy of the [KoBo Toolbox](http://www.kobotoolbox.org) survey data collection platform on a machine of your choosing. It relies on [Docker](https://docker.com) to separate the different parts of KoBo into different containers (which can be thought of as lighter-weight virtual machines) and [Docker Compose](https://docs.docker.com/compose/) to configure, run, and connect those containers. 


## Important notice when upgrading from commit `5c2ef02` (March 4, 2019) or earlier 

You must follow [these important instructions](./doc/March-2019-Upgrade.md). Then, please follow [next instructions](##important-notice-when-upgrading-from-todo-insert-final-1db-release-here-or-earlier)

If you do not, the application may not start or your data may not be visible.


## Important notice when upgrading from `[TODO: INSERT FINAL 1DB RELEASE HERE]` or earlier

If your version is older than [March 4, 2019](https://github.com/kobotoolbox/kobo-docker/commit/5c2ef0273339bee5c374830f72e52945947042a8 ""), please follow [these instructions]() **first**.

Up to and including release `[TODO: INSERT FINAL 1DB RELEASE HERE]`,
[KPI](https://github.com/kobotoolbox/kpi) and
[KoBoCAT](https://github.com/kobotoolbox/kobocat) both shared a common Postgres
database. They now each have their own, separate databases. Please continue
reading to learn how to migrate the (smaller) KPI tables to a new database and
adjust your configuration appropriately.

1. …
1. `postgres/master/clone_data_from_kc_to_kpi.sh`
1. …
1. Even though they are no longer used, it is not yet recommended to erase the
   KPI-only tables (e.g. `kpi_asset`) from KoBoCAT's Postgres database.
   Instructions for doing so will be included in a later release.

## Architecture

Below is a diagram (made with [Lucidchart](https://www.lucidchart.com)) of the containers that make up a running `kobo-docker` system and their connections.

![Diagram of Docker Containers](./doc/container-diagram.svg)

### Secure your installation
`kobo-docker` opens ports on all interfaces to let the `frontend` containers communicate with `backend`containers.  
A firewall is **HIGHLY recommended**. PostgreSQL, Redis and MongoDB ports must be closed when the server is exposed publicly.


## Setup procedure

This procedure has been simplified by using [`kobo-install`](https://github.com/kobotoolbox/kobo-install ""). 
Please use it to install `kobo-docker`.   

Already have an existing installation? Please see below.

1. **Migrating from RabbitMQ to Redis as the Celery (asynchronous task) broker**

    The easiest way is to rely on `kobo-install` to generate the correct environment files. 

    If you want to change it manually, edit:

    - `kobo-deployments/envfiles/kpi.txt`

    > `KPI_BROKER_URL=amqp://kpi:kpi@rabbit.[internal domain name]:5672/kpi` 

    to
    > `KPI_BROKER_URL=redis://redis-main.[internal domain name]:6389/1`

    - `kobo-deployments/envfiles/kobocat.txt`

    > `KOBOCAT_BROKER_URL=amqp://kobocat: kobocat@rabbit.[internal domain name]:5672/kobocat ` 

    to
    > `KOBOCAT_BROKER_URL =redis://redis-main.[internal domain name]:6389/2`

2. **Load balancing and redundancy**

    1. Load balancing
        `kobo-docker` has two different composer files. One for `frontend` and one for `backend`. 
    
        1. `frontend`: 
            - `NGinX`
            - `KoBoCat`
            - `KPI`
            - `Enketo Express`
    
        2. `backend`:
            - `PostgreSQL`
            - `MongoDB`
            - `redis`
    
        Docker-compose for `frontend` can be started on its own server, same thing for `backend`. Users can start as many `frontend` servers they want. A load balancer can spread the traffic between `frontend` servers. 
        `kobo-docker` uses (private) domain names between `frontend` and `backend`. 
        It's fully customizable in configuration files. Once again `kobo-install` does simplify the job by creating the configuration files for you.

    2. Redundancy
        `Backend` containers not redundant yet. Only `PostgreSQL` can be configured in `Master/Slave` mode where `Slave` is a real-time read-only replica. 
       
    This is a diagram shows how `kobo-docker` can be used for a load-balanced/(almost) redundant solution.
       
    _NB: The diagram is based on AWS infrastructure, but it's not required to host your environment there._

    ![aws diagram](./doc/aws-diagram.svg)

## Usage
It's recommended to create `*.override.yml` docker-compose files to customize your environment. It makes easier to update. 

- `docker-compose.frontend.override.yml`
- `docker-compose.backend.master.override.yml`
- `docker-compose.backend.slave.override.yml` (if a postgres replica is used)

1. **Start/start containers** 

    `$kobo-docker> docker-compose -f docker-compose.frontend.yml [-f docker-compose.frontend.override.yml] up -d`  
    `$kobo-docker> docker-compose -f docker-compose.backend.master.yml [-f docker-compose.backend.master.override.yml] up -d`
    `$kobo-docker> docker-compose -f docker-compose.frontend.yml [-f docker-compose.frontend.override.yml] stop`  
    `$kobo-docker> docker-compose -f docker-compose.backend.master.yml [-f docker-compose.backend.master.override.yml] stop`


2. **Backups**

    Automatic, periodic backups of KoBoCAT media, MongoDB, PostgreSQL and Redis can be individually enabled by uncommenting (and optionally customizing) the `*_BACKUP_SCHEDULE` variables in your envfiles. 
    
     - `deployments/envfiles/databases.txt` (MongoDB, PostgreSQL, Redis)
     - `deployments/envfiles/kobocat.txt` (KoBoCat media)
    
    When enabled, timestamped backups will be placed in backups/kobocat, backups/mongo, backups/postgres and backups/redis respectively.
    
    If `AWS` credentials and `AWS S3` backup bucket name are provided, the backups are created directly on `S3`.
    
    Backups **on disk** can also be manually triggered when kobo-docker is running by executing the the following commands:
    
    ```
    $kobo-docker> docker-compose -f docker-compose.frontend.yml [-f docker-compose.frontend.override.yml] exec kobocat /srv/src/kobocat/docker/backup_media.bash
    $kobo-docker> docker-compose -f docker-compose.backend.master.yml [-f docker-compose.backend.master.override.yml] exec mongo /bin/bash /kobo-docker-scripts/backup-to-disk.bash
    $kobo-docker> docker-compose -f docker-compose.backend.master.yml [-f docker-compose.backend.master.override.yml] exec -e PGUSER=kobo postgres /bin/bash /kobo-docker-scripts/backup-to-disk.bash
    $kobo-docker> docker-compose -f docker-compose.backend.master.yml [-f docker-compose.backend.master.override.yml] exec redis_main /bin/bash /kobo-docker-scripts/backup-to-disk.bash
    ```

2. **Restore backups**

    Commands should be run within containers.

     - MongoDB: `mongorestore --archive=<path/to/mongo.backup.gz> --gzip`
     - PostgreSQL: `pg_restore -U kobo -d kobotoolbox -c "<path/to/postgres.pg_dump>"`
     - Redis: `gunzip <path/to/redis.rdb.gz> && mv <path/to/extracted_redis.rdb> /data/enketo-main.rdb` 

3. **Maintenance mode**

    There is one composer file `docker-compose.maintenance.yml` can be used to put `KoBoToolbox` in maintenance mode.
    
    `nginx` container has to be stopped before launching the maintenance container.
    
    **Start** 
    
    ```
    docker-compose -f docker-compose.frontend.yml [-f docker-compose.frontend.override.yml] stop nginx
    docker-compose -f docker-compose.maintenance.yml up -d
    ``` 
    
    **Stop** 
    
    ```
    docker-compose -f docker-compose.maintenance.yml down
    docker-compose -f docker-compose.frontend.yml [-f docker-compose.frontend.override.yml] up -d nginx
    ```
    
    There are 4 variables that can be customized in `docker-compose.maintenance.yml`
    
    - `ETA` e.g. `2 hours`
    - `DATE_STR` e.g. `Monday, November 26 at 02:00 GMT`
    - `DATE_ISO` e.g. `20181126T02`
    - `EMAIL` e.g. `support@example.com`

## Warning

**These instructions are obselete and must be updated!**

If you started running KoBo Toolbox using a version of `kobo-docker` from before to [2016.10.13](https://github.com/kobotoolbox/kobo-docker/commit/316b1464c86e2c447ca88c10d383662b4f2e4ac6), actions that recreate the `kobocat` container (including `docker-compose up ...` under some circumstances) will result in losing access to user media files (e.g. responses to photo questions). Safely stored media files can be found in `kobo-docker/.vols/kobocat_media_uploads/`.

Files that were not safely stored can be found inside Docker volumes as well as in your current and any previous `kobocat` containers. One quick way of finding these is directly searching the Docker root directory. The root directory can be found by running `docker info` and looking for the "Docker Root Dir" entry (not to be confused with the storage driver's plain "Root Dir").

Once this is noted, you can `docker-compose stop` and search for potentially-missed media attachment directories with something like `sudo find ${YOUR_DOCKER_ROOT_DIR}/ -name attachments`. These attachment directories will be of the format `.../${SOME_KOBO_USER}/attachments` and you will want to back up each entire KoBo user's directory (the parent of the `attachments` directory) for safe keeping, then move/merge them under `.vols/kobocat_media_uploads/`, creating that directory if it doesn't exist and taking care not to overwrite any newer files present there if it does exist. Once this is done, clear out the old container and any attached volumes with `docker-compose rm -v kobocat`, then `git pull` the latest `kobo-docker` code, `docker-compose pull` the latest images for `kobocat` and the rest, and your media files will be safely stored from there on.

## Troubleshooting

- ### Basic troubleshooting
    You can confirm that your containers are running with `docker ps`. 
    To inspect the log output from:
     
     - the frontend containers, execute `docker-compose -f docker-compose.frontend.yml [-f docker-compose.frontend.override.yml] logs -f`
     - the master backend containers, execute `docker-compose -f docker-compose.backend.master.yml [-f docker-compose.backend.master.override.yml] logs -f`
     - the slaved backend container, execute `docker-compose -f docker-compose.backend.slave.yml [-f docker-compose.backend.slave.override.yml] logs -f`
       
    For a specific container use e.g. `docker-compose -f docker-compose.backend.master.yml [-f docker-compose.backend.master.override.yml] logs -f redis_main`.
    
    The documentation for Docker can be found at https://docs.docker.com.

- ### Django debugging
Developers can use [PyDev](http://www.pydev.org/)'s [remote, graphical Python debugger](http://www.pydev.org/manual_adv_remote_debugger.html) to debug Python/Django code. To enable for the `kpi` container:

    1. Specify the mapping(s) between target Python source/library paths on the debugging machine to the locations of those files/directories inside the container by customizing and uncommenting the `KPI_PATH_FROM_ECLIPSE_TO_PYTHON_PAIRS` variable in [`envfiles/kpi.txt`](./envfiles/kpi.txt).
    2. Share the source directory of the PyDev remote debugger plugin into the container by customizing (taking care to note the actual location of the version-numbered directory) and uncommenting the relevant `volumes` entry in your `docker-compose.yml`.
    3. To ensure PyDev shows you the same version of the code as is being run in the container, share your live version of any target Python source/library files/directories into the container by customizing and uncommenting the relevant `volumes` entry in your `docker-compose.yml`.
    4. Start the PyDev remote debugger server and ensure that no firewall or other settings will prevent the containers from connecting to your debugging machine at the reported port.
    5. Breakpoints can be inserted with e.g. `import pydevd; pydevd.settrace('${DEBUGGING_MACHINE_IP}')`.

    Remote debugging in the `kobocat` container can be accomplished in a similar manner.
    

## Redis performance
Please take a look at [https://www.techandme.se/performance-tips-for-redis-cache-server/](https://www.techandme.se/performance-tips-for-redis-cache-server/)
to get rid of Warning message when starting redis containers

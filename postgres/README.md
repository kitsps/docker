OpenBMP Postgres Container
----------------------------
This container provides PostgreSQL backend to OpenBMP. It requires the following other containers:

- **openbmp/collector**
- **openbmp/kafka** or some Kafka instance.  Does not have to be the OpenBMP one. 

#### Container Includes the following
- **Postgres** - Latest postgres 10.x release. TCP port 5432
- **TimescaleDB** - Latest version of TimescaleDB
- **RPKI Validator** - RPKI Validator 2.24. TCP port 8080 

### Kafka Validation Testing
The Kafka setup can be tricky due to docker networking between containers and remote systems. Kafka clustering
makes use of a bootstrap server which will advertise each broker ```hostname:port``` that the consumer/producer
will use.  Each consumer/producer will connect to the brokers using these **advertised** hostnames and ports.  The
setting in Kafka to configure the broker hostname is ```advertised.listeners```.  The [OpenBMP Kafka](https://github.com/OpenBMP/docker/tree/master/kafka)
container sets the ```advertised.listeners``` to the environment variable value of **KAFKA_FQDN**:9092.

> NOTE: the port is currently static at 9092 due to NAT/PAT not working well with Kafka advertised listeners and docker container port mapping.

The postgres container (**this container**) uses the **KAFKA_FQDN** as the bootstrap server.  This will work with an
IP or hostname. When using a hostname, the hostname *MUST* resolve within the container.  While this may work for
bootstrap server conection, the advertised hostnames need to also resolve in the container.  If using the OpenBMP
Kafka container, the same KAFKA_FQDN should work fine.   

Regardless of using your own Kafka install or the OpenBMP Kafka container, the postgres container will attempt
to produce and consume messages to Kafka as a validation test.  This will ensure that the bootstrap and at least one broker server is
working. If the docker container quits after starting, check ```docker logs openmbp_psql``` for errors relating to
Kafka communication.  These will need to be resolved before proceeding. 


**Kafka Validation is a 3 step process** 

1. Successfully connect to the bootstrap server and retrieve metadata (e.g.  broker hostname:port)
2. Successfully produce a test message to ```openbmp.parsed.test``` topic
3. Successfully consume a test message from ```openbmp.parsed.test``` topic

> #### IMPORTANT
> If using your own Kafka install, make sure you allow producing/consuming to/from **openbmp.parsed.test** 
> for the consumer validation. 

#### Hostnames in Container
You can map the Kafka hostname and each broker if they are different using two methods:

1. add ```--add-host HOSTNAME:IP``` to **docker run** command.  Make sure to add one for the bootstrap and each broker.  
2. Create a **/var/openbmp/config/hosts** file and add the Kafka bootstrap and broker hostname to IP mappings. 

> #### IMPORTANT
> Docker networking is a pain sometimes, but you can use the **docker0** IP for connections between containers.  If
> you are using the OpenBMP Kafka container and the Postgres container on the same host, then use the **docker0** IP 
> address for the Kafka bootstrap/broker hostname mapping.  This IP by default should be **172.17.0.1**.  Check your 
> install for the correct IP to use. 

### Recommended Current Linux Distributions and system requirements

  1. Ubuntu 18.04/Bionic
  1. CentOS 7/RHEL 7

#### Storage

You will need to dedicate space for the postgres instance.  Normally two partitions are used.  A good
starting size for postgres main is 500GB and postgres ts (timescaleDB) is 1TB.  Both disks
should be fast SSD. ZFS can be used on either of them to add compression. The size you need will depend
on the number of NLRI's and updates per second.

#### Memory & CPU

The size of memory will depend on the type of queries and number of NLRI's.   A good starting point for
memory is a server with more than 48GB RAM. You can run on as little as 4GB RAM but that will only
scale to about 10,000,000 NLRI's.  64BG of RAM should scale to 150,000,000 NLRI's. 

The number of vCPU's also varies by the number of concurrent connections and how many threads you use for
the postgres consumer.  A good starting point is at least 8 vCPU's.   

##### NOTE: Postgres can be killed by the Linux OOM-Killer
This is very bad as it causes Postgres to restart. This will happen because postgres uses a large shared buffer,
which causes the OOM to believe it's using a lot of VM.     

It is suggested to run the postgres server with the following Linux settings:

    # Update runtime
    sysctl -w vm.vfs_cache_pressure=500
    sysctl -w vm.swappiness=10
    sysctl -w vm.min_free_kbytes=1000000
    sysctl -w vm.overcommit_memory=2
    sysctl -w vm.overcommit_ratio=95   

    # Update startup    
    echo "vm.vfs_cache_pressure=500" >> /etc/sysctl.conf
    echo "vm.min_free_kbytes=1000000" >> /etc/sysctl.conf
    echo "vm.swappiness=10" >> /etc/sysctl.conf
    echo "vm.overcommit_memory=2" >> /etc/sysctl.conf
    echo "vm.overcommit_ratio=95" >> /etc/sysctl.conf


See Postgres [hugepages](https://www.postgresql.org/docs/current/static/kernel-resources.html#LINUX-HUGE-PAGES) for
details on how to enable and use hugepages.   Some Linux distributions enable **transparent hugepages** which
will prevent the ability to configure ```vm.nr_hugepages```. If you find that you cannot set ```vm.nr_hugepages```,
then try the below:

    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    echo never > /sys/kernel/mm/transparent_hugepage/defrag
    sync && echo 3 > /proc/sys/vm/drop_caches


#### Postgres Vacuum (reclaim disk space)
Postgres reclaims deleted/updated records using the vacuum process.  You can run this manually/cron via the 
```VACUUM``` command.  **autovacuum** is used to do this periodically.   Careful tuning of this
is required.  Checkout [autovacuum-tuning-basics](https://blog.2ndquadrant.com/autovacuum-tuning-basics/), 
[Routine Vacuuming](https://www.postgresql.org/docs/current/static/routine-vacuuming.html), and
[VACUUM](https://www.postgresql.org/docs/current/static/sql-vacuum.html) for more details. 


### 1) Install docker
Follow the [Docker Instructions](https://docs.docker.com/install) to install docker CE.  

- - -

### 2) Add persistent volumes

Persistent volumes make it possible for upgrades with out loosing any data. 

#### (a) Create persistent config location

    mkdir -p /var/openbmp/config
    chmod 777 /var/openbmp/config

##### config/hosts
You can add custom host entries so that the collector will reverse lookup IP addresses
using a persistent hosts file.

Run docker with ```-v /var/openbmp/config:/config``` to make use of the persistent config files.

##### config/obmp-psql.yml
If the [obmp-psql.yml](https://github.com/OpenBMP/obmp-postgres/blob/master/src/main/resources/obmp-psql.yml) file
does not exist, a default one will be created. You should update this based on your settings. This file
is inline documented.  

#### (b) Create persistent postgres locations

*You should use fast SSD and/or ZFS.*  Size of these locations/mount points are directly related to the 
number of NLRI's maintained and number of changes/updates per second. 

> TODO: Will post numbers of how to determine the disk size needed.  For now, if you have less
> than 50,000,00 prefixes, then you can use 1TB.  If you have more than that, you should consider
> multiple disks.  ZFS can make your life easier as you can easily add disks and it supports compression.    

- **postgres/main** - This location will be used for the main postgres data
files and tables. 

> This really should be a mount point to a dedicated filesystem

```
    mkdir -p /var/openbmp/postgres/main
    chmod 7777 /var/openbmp/postgres/main
``` 

- **postgres/ts** - This location will be used for the time series postgres tables

> This really should be a mount point to a dedicated filesystem

```
    mkdir -p /var/openbmp/postgres/ts
    chmod 7777 /var/openbmp/postgres/ts
```

### 3) Run docker container

> Running the docker container for the first time will download the container image. 

#### Environment Variables
Below table lists the environment variables that can be used with ``docker run -e <name=value>``

NAME | Value | Details
:---- | ----- |:-------
KAFKA\_FQDN | hostanme or IP | Kafka broker hostname.  Hostname can be an IP address.
ENABLE_RPKI | 1 | Set to 1 to eanble RPKI. RPKI is disabled by default
ENABLE_IRR | 1 | Set to 1 to enable IRR. IRR is disabled by default
MEM | number | Number value in GB to allocate to Postgres.  This will be the shared_buffers value.
PGUSER | username | Postgres username, default is **openbmp**
PGPASSWORD | password | Postgres password, default is **openbmp**
PGDATABASE | database | Name of postgres database, default is **openbmp**


#### Run normally

> ##### NOTE:
> If the container fails to start, it's likely due to the configuration. Check using
> ```docker logs openbmp_psql```

```
docker run -d --name openbmp_psql \
	-h obmp-psql \
	-e ENABLE_RPKI=1 \
	-e ENABLE_IRR=1 \
	-e KAFKA_FQDN=kafka.domain \
	--add-host kafka.domain:172.17.0.1 \
	-e MEM=16 \
	--shm-size=512m \
	-v /var/openbmp/config:/config \
	-v /var/openbmp/postgres/main:/data/main \
	-v /var/openbmp/postgres/ts:/data/ts \
	-p 5432:5432 -p 9005:9005 -p 8080:8080 \
	openbmp/postgres
```

> The above uses **kafka.domain** and maps that in the container to the **docker0** IP. This 
> ensures that the kafka hostname will resolve correctly within the container for the
> openbmp/kafka instance that runs on the same host. Change this to your Kafka bootstrap
> server.  Make sure that all broker hostnames can be resolved by the container. When
> in doubt, use **--add-host** or add entries to the **/var/openbmp/config/hosts** file. 

### Monitoring/Troubleshooting

Useful commands:

- docker logs openbmp_psql
- docker exec openbmp_psql tail -f /var/log/obmp-psql.log
- docker exec openbmp_psql tail -f /var/log/postgresql/postgresql-10-main.log 
- docker exec -it openbmp_psql bash


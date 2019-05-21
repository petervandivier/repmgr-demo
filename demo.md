<!-- 
.SOURCEURL
    https://blog.raveland.org/post/postgresql_repmgr_pgbouncer_en/
-->

# Setup a PostgreSQL cluster with repmgr and pgbouncer
Recently I had to setup a PostgreSQL cluster and one of the prerequisites was to use repmgr.

In this post, I will explain you the work I did and how to setup this kind of cluster.

For the curious, all the documentation for repmgr is available [here][1].

## The architecture
My setup is:

* node0 / PostgreSQL (192.168.1.50)
* node1 / PostgreSQL (192.168.1.51)
* node2 / PostgreSQL (192.168.1.52)
* pgbouncer / pgbouncer (192.168.1.53)

node0 to node2 are the PostgreSQL machines with one primary and two secondaries.

The pgbouncer machine will receive all the queries and send them to the actual primary.

## Setup the machines
For this post, all my machines will be on Debian 9 with PostgreSQL 10.

### Install the packages
* Install the 2ndQuadrant repository (for repmgr and pgbouncer) on **all the PostgreSQL machines**

```sh
apt-get install apt-transport-https
echo "deb https://dl.2ndquadrant.com/default/release/apt stretch-2ndquadrant main" > /etc/apt/sources.list.d/2ndquadrant.list
wget --quiet -O - https://dl.2ndquadrant.com/gpg-key.asc | apt-key add -
```

* Install PostgreSQL and repmgr on **all the PostgreSQL machines**

```sh
apt-get install postgresql-10 postgresql-10-repmgr
systemctl stop postgresql
```

We need to stop PostgreSQL because we have to do some configuration.

* Install the PostgreSQL repository (for PostgreSQL 10) **on all the machines (PostgreSQL and pgbouncer)**

```sh
echo "deb http://apt.postgresql.org/pub/repos/apt/ stretch-pgdg main" > /etc/apt/sources.list.d/pgdg.list
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
apt-get update
```

* Install pgbouncer and the pgsql client on the pgbouncer machine

```sh
apt-get install pgbouncer postgresql-client-10
```

### Setup SSH passwordless
Passwordless SSH connectivity between all machines is not mandatory but is required for these cases:

* `repmgr standby switchover` [(documentation)][2]
* `repmgr cluster matrix` [(documentation)][3]
* `repmgr cluster crosscheck` [(documentation)][4]

We will use it for pgbouncer too (as you can see later).

Follow these steps:

* Create a new pair of ssh keys and save them in the temporary directory

```sh
$ % ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/pea/.ssh/id_rsa): /home/pea/tmp/repmgr/id_rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/pea/tmp/repmgr/id_rsa.
Your public key has been saved in /home/pea/tmp/repmgr/id_rsa.pub.
```

* Create an `.ssh` directory on all machines (PostgreSQL and pgbouncer) for the postgres user:

```sh
for i in node0 node1 node2 pgbouncer ; 
do 
  echo ${i} ; 
  ssh root@${i} "mkdir /var/lib/postgresql/.ssh && chown postgres:postgres /var/lib/postgresql/.ssh" ; 
done
```

* Copy the ssh keys on all the machines (PostgreSQL and pgbouncer)

```sh
for i in node0 node1 node2 pgbouncer ; 
do 
  echo ${i} ; 
  scp id_rsa* root@${i}:/var/lib/postgresql/.ssh/ ; 
done

for i in node0 node1 node2 pgbouncer ; 
do 
  echo ${i} ; 
  scp id_rsa.pub root@${i}:/var/lib/postgresql/.ssh/authorized_keys ; 
done
```

* Generate the file `known_hosts` and copy it to all machines

```sh
ssh-keyscan node0.raveland.priv node1.raveland.priv node2.raveland.priv pgbouncer.raveland.priv > known_hosts

for i in node0 node1 node2 pgbouncer ; 
do 
  echo ${i} ; 
  scp known_hosts root@${i}:/var/lib/postgresql/.ssh/ ; 
done

for i in node0 node1 node2 pgbouncer ; 
do 
  echo ${i} ; 
  ssh root@${i} "chown postgres:postgres /var/lib/postgresql/.ssh/*" ; 
done
```

**WARNING**: for the last step, I used the fqdn of my machines. It’s better to do like this to avoid some problems.

Now we have a SSH passwordless connectivity between all machines for the _postgres_ user.

## Configuring repmgr
In order to configure repmgr, you will need to:

* know the network subnet where repmgr will be deployed
* update the PostgreSQL configuration (file _postgresql.conf_)
* update the PostgreSQL client authentication configuration file (file _pg_hba.conf_)

In order to bootstrap your cluster, you will have to choose a server that will be the primary.

In my case, the network subnet is : `192.168.1.0/24` and the primary will be `node0`.

**HINT**: all the repmgr commands must be run as user _postgres_.

### Changes to apply to all PostgreSQL machines
These changes must be applied on all the PostgreSQL machines (both primary and secondaries).

At first, we need to configure _sudo_.

* Create the file `/etc/sudoers.d/postgres`

```sh
echo "postgres ALL = NOPASSWD: /usr/bin/pg_ctlcluster" > /etc/sudoers.d/postgres
```

Then we will modify the `/etc/postgresql/10/main/postgresql.conf` file.

* Change `listen_addresses` to `listen_addresses = '*'`
* Change `shared_preload_libraries` to `shared_preload_libraries = 'repmgr'`
* Add at the end of the file, this line: `include 'postgresql.replication.conf'`
* Create the file `/etc/postgresql/10/main/postgresql.replication.conf` with this:

```properties
max_wal_senders       = 15
max_replication_slots = 15
wal_level             = 'replica'
hot_standby           = on
archive_mode          = on
archive_command       = '/bin/true'
wal_keep_segments     = 500
```

Of course, you can adjust these values to your setup.

* Now we need to adapt the `/etc/postgresql/10/main/pg_hba.conf` file. Add the following lines

```properties
local   replication   repmgr                              trust
host    replication   repmgr      127.0.0.1/32            trust
host    replication   repmgr      192.168.1.0/24          trust

local   repmgr        repmgr                              trust
host    repmgr        repmgr      127.0.0.1/32            trust
host    repmgr        repmgr      192.168.1.0/24          trust
```

On Debian, add these lines just after this one : `local all postgres peer`

Of course, change the subnet `192.168.1.0/24` by yours.

**WARNING**: you need to adjust `pg_hba.conf` to allow your applications to connect to the PostgreSQL server.

For this setup, I added at the end: `host all all 192.168.1.0/24 md5`

### Configuring the primary PostgreSQL server
Now that you applied the changes above, follow these steps:

* Start PostgreSQL: `# systemctl start postgresql`
* Create the repmgr database and user:

```sh
sudo su - postgres
$ createuser -s repmgr
$ createdb repmgr -O repmgr
```

* Create the file `/etc/repmgr.conf` (the _postgres_ must be able to read it)

```properties
node_id=1
node_name="node0.raveland.priv"
conninfo='host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/10/main'

use_replication_slots=yes
monitoring_history=yes

service_start_command   = 'sudo /usr/bin/pg_ctlcluster 10 main start'
service_stop_command    = 'sudo /usr/bin/pg_ctlcluster 10 main stop'
service_restart_command = 'sudo /usr/bin/pg_ctlcluster 10 main restart'
service_reload_command  = 'sudo /usr/bin/pg_ctlcluster 10 main reload'
service_promote_command = 'sudo /usr/bin/pg_ctlcluster 10 main promote'

promote_check_timeout = 15

failover=automatic
promote_command='/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='/usr/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'

log_file='/var/log/postgresql/repmgrd.log'
```

In this file, only 4 parameters are mandatory:

* node_id
* node_name
* conninfo
* data_directory

You can find a documented _repmgr.conf_ [here][5].

**WARNING 1**: for the Debian users, it’s recommended to use the wrapper _pg_ctlcluster_ to control PostgreSQL.

**WARNING 2**: in this configuration, i already setup some instructions for _repmgrd_.

* Register the primary server:

```sh
sudo su - Postgres
$ repmgr -f /etc/repmgr.conf primary register
INFO: connecting to primary database...
NOTICE: attempting to install extension "repmgr"
NOTICE: "repmgr" extension successfully installed
NOTICE: primary node record (id: 1) registered
```

Now our primary server is up and running.

### Configuring the secondaries PostgreSQL servers
Follow these steps to configure each of your secondaries servers:

* Ensure that PostgreSQL is shutdown: `systemct stop postgresql`
* Trash the data directory of PostgreSQL: `rm -fR /var/lib/postgresql/10/main/*`
* Ensure you can connect to the primary node: `psql -h node0.raveland.priv -U repmgr`
* Create the `/etc/repmgr.conf` file like as on the primary server but adapt the values `node_id`, `node_name` and `conninfo`

```properties
node_id=2
node_name="node1.raveland.priv"
conninfo='host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/10/main'

use_replication_slots=yes
monitoring_history=yes

service_start_command   = 'sudo /usr/bin/pg_ctlcluster 10 main start'
service_stop_command    = 'sudo /usr/bin/pg_ctlcluster 10 main stop'
service_restart_command = 'sudo /usr/bin/pg_ctlcluster 10 main restart'
service_reload_command  = 'sudo /usr/bin/pg_ctlcluster 10 main reload'
service_promote_command = 'sudo /usr/bin/pg_ctlcluster 10 main promote'

promote_check_timeout = 15

failover=automatic
promote_command='/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='/usr/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'

log_file='/var/log/postgresql/repmgrd.log'
```

* Clone the primary server (first dry-run)

```sh
sudo su - PostgreSQL
$ repmgr -h node0.raveland.priv -U repmgr -d repmgr -f /etc/repmgr.conf standby clone --dry-run
NOTICE: destination directory "/var/lib/postgresql/10/main" provided
INFO: connecting to source node
DETAIL: connection string is: host=node0.raveland.priv user=repmgr dbname=repmgr
DETAIL: current installation size is 30 MB
NOTICE: standby will attach to upstream node 1
HINT: consider using the -c/--fast-checkpoint option
INFO: all prerequisites for "standby clone" are met
```

* Clone the primary server (for real this time)

```sh
$ repmgr -h node0.raveland.priv -U repmgr -d repmgr -f /etc/repmgr.conf standby clone
NOTICE: destination directory "/var/lib/postgresql/10/main" provided
INFO: connecting to source node
DETAIL: connection string is: host=node0.raveland.priv user=repmgr dbname=repmgr
DETAIL: current installation size is 30 MB
INFO: checking and correcting permissions on existing directory "/var/lib/postgresql/10/main"
NOTICE: starting backup (using pg_basebackup)...
HINT: this may take some time; consider using the -c/--fast-checkpoint option
INFO: executing:
  pg_basebackup -l "repmgr base backup"  -D /var/lib/postgresql/10/main -h node0.raveland.priv -p 5432 -U repmgr -X stream -S repmgr_slot_2
NOTICE: standby clone (using pg_basebackup) complete
NOTICE: you can now start your PostgreSQL server
HINT: for example: sudo /usr/bin/pg_ctlcluster 10 main start
HINT: after starting the server, you need to register this standby with "repmgr standby register"
```

* Start the PostgreSQL server

```sh
$ sudo /usr/bin/pg_ctlcluster 10 main start
```

* Register the secondary

```sh
$ repmgr -f /etc/repmgr.conf standby register
INFO: connecting to local node ""node1.raveland.priv"" (ID: 2)
INFO: connecting to primary database
WARNING: --upstream-node-id not supplied, assuming upstream node is primary (node ID 1)
INFO: standby registration complete
NOTICE: standby node ""node1.raveland.priv"" (id: 2) successfully registered
```

* Do the same for all your secondaries
* As an example, here is the steps for my node2:
    * `# systemct stop postgresql`
    * `# rm -fr /var/lib/postgresql/10/main/*`
    * Create `/etc/repmgr.conf`

```properties
node_id=3
node_name="node2.raveland.priv"
conninfo='host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/10/main'

use_replication_slots=yes
monitoring_history=yes

service_start_command   = 'sudo /usr/bin/pg_ctlcluster 10 main start'
service_stop_command    = 'sudo /usr/bin/pg_ctlcluster 10 main stop'
service_restart_command = 'sudo /usr/bin/pg_ctlcluster 10 main restart'
service_reload_command  = 'sudo /usr/bin/pg_ctlcluster 10 main reload'
service_promote_command = 'sudo /usr/bin/pg_ctlcluster 10 main promote'

promote_check_timeout = 15

failover=automatic
promote_command='/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file'
follow_command='/usr/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n'

log_file='/var/log/postgresql/repmgrd.log'
```

* Clone the primary server

```sh
$ repmgr -h node0.raveland.priv -U repmgr -d repmgr -f /etc/repmgr.conf standby clone
NOTICE: destination directory "/var/lib/postgresql/10/main" provided
INFO: connecting to source node
DETAIL: connection string is: host=node0.raveland.priv user=repmgr dbname=repmgr
DETAIL: current installation size is 30 MB
INFO: checking and correcting permissions on existing directory "/var/lib/postgresql/10/main"
NOTICE: starting backup (using pg_basebackup)...
HINT: this may take some time; consider using the -c/--fast-checkpoint option
INFO: executing:
  pg_basebackup -l "repmgr base backup"  -D /var/lib/postgresql/10/main -h node0.raveland.priv -p 5432 -U repmgr -X stream -S repmgr_slot_3
NOTICE: standby clone (using pg_basebackup) complete
NOTICE: you can now start your PostgreSQL server
HINT: for example: sudo /usr/bin/pg_ctlcluster 10 main start
HINT: after starting the server, you need to register this standby with "repmgr standby register"
```

* Start PostgreSQL: `/usr/bin/pg_ctlcluster 10 main start`
* Register the secondary

```sh
$ repmgr -f /etc/repmgr.conf standby register
INFO: connecting to local node ""node2.raveland.priv"" (ID: 3)
INFO: connecting to primary database
WARNING: --upstream-node-id not supplied, assuming upstream node is primary (node ID 1)
INFO: standby registration complete
NOTICE: standby node ""node2.raveland.priv"" (id: 3) successfully registered
```

## Playing with repmgr
Now all our servers are up and running and we have a running PostgreSQL cluster.

We will see how to do basic operations with repmgr.

**REMINDER**: all the repmgr commands must be run as postgres user.

### Cluster status

```sh
  $ repmgr -f /etc/repmgr.conf cluster show
  ID | Name                  | Role    | Status    | Upstream              | Location | Connection string                                                   
  ----+-----------------------+---------+-----------+-----------------------+----------+----------------------------------------------------------------------
  1  | "node0.raveland.priv" | primary | * running |                       | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
  2  | "node1.raveland.priv" | standby |   running | "node0.raveland.priv" | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
  3  | "node2.raveland.priv" | standby |   running | "node0.raveland.priv" | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
```

### My primary server has failed, what’s happen ?
Now we will see what to do when your secondary will stop for an unknown reason.

* Shutdown primary (simulate failure)

```sh
  systemctl stop Postgresql    
```

* Verify on secondary

```sh
  $ repmgr -f /etc/repmgr.conf cluster show
  ID | Name                  | Role    | Status        | Upstream              | Location | Connection string                                                   
  ----+-----------------------+---------+---------------+-----------------------+----------+----------------------------------------------------------------------
  1  | "node0.raveland.priv" | primary | ? unreachable |                       | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
  2  | "node1.raveland.priv" | standby |   running     | "node0.raveland.priv" | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
  3  | "node2.raveland.priv" | standby |   running     | "node0.raveland.priv" | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2

  WARNING: following issues were detected
   unable to connect to node ""node0.raveland.priv"" (ID: 1)
   node ""node0.raveland.priv"" (ID: 1) is registered as an active primary but is unreachable
```

* Promote node1 as primary

```sh
  $ repmgr -f /etc/repmgr.conf standby promote
  NOTICE: promoting standby to primary
  DETAIL: promoting server ""node1.raveland.priv"" (ID: 2) using "sudo /usr/bin/pg_ctlcluster 10 main promote"
  DETAIL: waiting up to 15 seconds (parameter "promote_check_timeout") for promotion to complete
  NOTICE: STANDBY PROMOTE successful
  DETAIL: server ""node1.raveland.priv"" (ID: 2) was successfully promoted to primary
```

* Verify on node2

```sh
  $ repmgr -f /etc/repmgr.conf cluster show
  ID | Name                  | Role    | Status               | Upstream              | Location | Connection string                                                   
  ----+-----------------------+---------+----------------------+-----------------------+----------+----------------------------------------------------------------------
  1  | "node0.raveland.priv" | primary | ? unreachable        |                       | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
  2  | "node1.raveland.priv" | standby | ! running as primary | "node0.raveland.priv" | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
  3  | "node2.raveland.priv" | standby |   running            | "node0.raveland.priv" | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2

  WARNING: following issues were detected
    unable to connect to node ""node0.raveland.priv"" (ID: 1)
    node ""node0.raveland.priv"" (ID: 1) is registered as an active primary but is unreachable
    node ""node1.raveland.priv"" (ID: 2) is registered as standby but running as primary
```

* That’s **not good** because node2 is still following _node0_.

* Update node2

```sh
  $ repmgr -f /etc/repmgr.conf standby follow
  NOTICE: setting node 3s primary to node 2
  NOTICE: restarting server using "sudo /usr/bin/pg_ctlcluster 10 main restart"
  WARNING: unable to connect to old upstream node 1 to remove replication slot
  HINT: if reusing this node, you should manually remove any inactive replication slots
  NOTICE: STANDBY FOLLOW successful
  DETAIL: standby attached to upstream node ""node1.raveland.priv"" (node ID: 2)
```

* Verify

```sh
  $ repmgr -f /etc/repmgr.conf cluster show
  ID | Name                  | Role    | Status    | Upstream              | Location | Connection string                                                   
  ----+-----------------------+---------+-----------+-----------------------+----------+----------------------------------------------------------------------
  1  | "node0.raveland.priv" | primary | - failed  |                       | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
  2  | "node1.raveland.priv" | primary | * running |                       | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
  3  | "node2.raveland.priv" | standby |   running | "node1.raveland.priv" | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2

  WARNING: following issues were detected
    unable to connect to node ""node0.raveland.priv"" (ID: 1)
```

Now we are fine (the old primary is still stopped). But as you can see it’s a bit complicated..

### My old primary is back.. what will happen ??
One thing to know about repmgr is that when an old primary comes back, i will **stay primary** !

* Restart the old primary (on node0): `systemctl start postgresql`
* Verify the status on the old primary (_node0_)

```sh
$ repmgr -f /etc/repmgr.conf cluster show
ID | Name                  | Role    | Status               | Upstream              | Location | Connection string                                                   
----+-----------------------+---------+----------------------+-----------------------+----------+----------------------------------------------------------------------
1  | "node0.raveland.priv" | primary | * running            |                       | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
2  | "node1.raveland.priv" | standby | ! running as primary | "node0.raveland.priv" | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
3  | "node2.raveland.priv" | standby |   running            | "node0.raveland.priv" | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2

WARNING: following issues were detected
   node ""node1.raveland.priv"" (ID: 2) is registered as standby but running as primary
```

It still thinks it’s a primary server but it detects that something goes wrong.

* Verify the status on secondaries

```sh
$ repmgr -f /etc/repmgr.conf cluster show
ID | Name                  | Role    | Status    | Upstream              | Location | Connection string                                                   
----+-----------------------+---------+-----------+-----------------------+----------+----------------------------------------------------------------------
1  | "node0.raveland.priv" | primary | ! running |                       | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
2  | "node1.raveland.priv" | primary | * running |                       | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
3  | "node2.raveland.priv" | standby |   running | "node1.raveland.priv" | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2

WARNING: following issues were detected
   node ""node0.raveland.priv"" (ID: 1) is running but the repmgr node record is inactive
```

The secondaries see the old primary but as inactive.

Here are the steps to follow to switch your old primary as a new secondary.

* Shutdown PostgreSQL on the old primary : `systemctl stop postgresql`
* Rejoin the node:

```sh
$ repmgr -f /etc/repmgr.conf node service --action=stop --checkpoint
$ repmgr -f /etc/repmgr.conf -d 'host=node1.raveland.priv user=repmgr dbname=repmgr' node rejoin
NOTICE: setting node 1's primary to node 2
NOTICE: starting server using "sudo /usr/bin/pg_ctlcluster 10 main start"
NOTICE: replication slot "repmgr_slot_2" deleted on node 1
WARNING: 1 inactive replication slots detected
DETAIL: inactive replication slots:
   repmgr_slot_3 (physical)
HINT: these replication slots may need to be removed manually
NOTICE: NODE REJOIN successful
DETAIL: node 1 is now attached to node 2
```

Of course, in the `host=` parameter, you must specify the address of the new primary.

### I want to do a maintenance on my primary server. How can i do ?
As you can see, when you lose a primary, you will have some manual operations to do.

There is an easier way to switch primary server : `switchover`.

Remember, now the primary is _node1_ on our cluster. But we need to shutdown it for maintenance.

* We want to promote _node2_ as a new primary. At first try the dry-run:

```sh
$ repmgr standby switchover -f /etc/repmgr.conf --siblings-follow --dry-run
NOTICE: checking switchover on node ""node2.raveland.priv"" (ID: 3) in --dry-run mode
INFO: SSH connection to host "node1.raveland.priv" succeeded
INFO: able to execute "repmgr" on remote host "localhost"
INFO: all sibling nodes are reachable via SSH
INFO: 2 walsenders required, 15 available
INFO: demotion candidate is able to make replication connection to promotion candidate
INFO: 0 pending archive files
INFO: replication lag on this standby is 0 seconds
INFO: 2 replication slots required, 15 available
NOTICE: local node ""node2.raveland.priv"" (ID: 3) would be promoted to primary; current primary ""node1.raveland.priv"" (ID: 2) would be demoted to standby
INFO: following shutdown command would be run on node ""node1.raveland.priv"":
  "sudo /usr/bin/pg_ctlcluster 10 main stop"
```

Everything looks good !

* Do the switchover:

```sh
$ repmgr standby switchover -f /etc/repmgr.conf --siblings-follow
NOTICE: executing switchover on node ""node2.raveland.priv"" (ID: 3)
NOTICE: local node ""node2.raveland.priv"" (ID: 3) will be promoted to primary; current primary ""node1.raveland.priv"" (ID: 2) will be demoted to standby
NOTICE: stopping current primary node ""node1.raveland.priv"" (ID: 2)
NOTICE: issuing CHECKPOINT
DETAIL: executing server command "sudo /usr/bin/pg_ctlcluster 10 main stop"
INFO: checking for primary shutdown; 1 of 60 attempts ("shutdown_check_timeout")
NOTICE: current primary has been cleanly shut down at location 0/7000028
NOTICE: promoting standby to primary
DETAIL: promoting server ""node2.raveland.priv"" (ID: 3) using "sudo /usr/bin/pg_ctlcluster 10 main promote"
DETAIL: waiting up to 15 seconds (parameter "promote_check_timeout") for promotion to complete
NOTICE: STANDBY PROMOTE successful
DETAIL: server ""node2.raveland.priv"" (ID: 3) was successfully promoted to primary
NOTICE: setting node 2s primary to node 3
NOTICE: starting server using "sudo /usr/bin/pg_ctlcluster 10 main start"
NOTICE: replication slot "repmgr_slot_3" deleted on node 2
WARNING: 1 inactive replication slots detected
DETAIL: inactive replication slots:
    repmgr_slot_1 (physical)
HINT: these replication slots may need to be removed manually
NOTICE: NODE REJOIN successful
DETAIL: node 2 is now attached to node 3
NOTICE: executing STANDBY FOLLOW on 1 of 1 siblings
INFO: STANDBY FOLLOW successfully executed on all reachable sibling nodes
NOTICE: switchover was successful
DETAIL: node ""node2.raveland.priv"" is now primary and node ""node1.raveland.priv"" is attached as standby
NOTICE: STANDBY SWITCHOVER has completed successfully
```

* Verify :

```sh
$ repmgr -f /etc/repmgr.conf cluster show
ID | Name                  | Role    | Status    | Upstream              | Location | Connection string                                                   
----+-----------------------+---------+-----------+-----------------------+----------+----------------------------------------------------------------------
1  | "node0.raveland.priv" | standby |   running | "node2.raveland.priv" | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
2  | "node1.raveland.priv" | standby |   running | "node2.raveland.priv" | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
3  | "node2.raveland.priv" | primary | * running |                       | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
```

Enjoy !!! 

## Introducing repmgrd: replication manager daemon
Now that you have your PostgreSQL cluster with repmgr, you are happy. But maybe you would like to have a bit of automation.. That’s where repmgrd comes !

As well said in the documentation: repmgrd is a management and monitoring daemon which runs on each node in a replication cluster.

It can automate actions such as failover and updating standbys to follow the new primary, as well as providing monitoring information about the state of each standby.

### Configure repmgrd
In this setup, you will just need to modify the file `/etc/default/repmgrd` to enable repmgrd at boot.

* Edit the file `/etc/default/repmgrd`:

```properties
REPMGRD_ENABLED=yes
REPMGRD_CONF="/etc/repmgr.conf"
REPMGRD_OPTS="--daemonize=false"
REPMGRD_USER=postgres
REPMGRD_BIN=/usr/bin/repmgrd
REPMGRD_PIDFILE=/var/run/repmgrd.pid
```

* Start the daemon: `systemctl restart repmgrd` (yes restart)
* watch the logs:

```sh
$ tail /var/log/postgresql/repmgrd.log
[2018-12-21 21:54:02] [NOTICE] starting monitoring of node ""node0.raveland.priv"" (ID: 1)
[2018-12-21 21:54:02] [INFO] monitoring connection to upstream node ""node2.raveland.priv"" (node ID: 3)
```

* Do this on all the PostgreSQL machines

### Playing with repmgrd

#### The primary fails
Let see what happens when the primary fails for an unknown reason (like the example above).

* Shutdown the primary (simulate failure): `systemctl stop postgresql`
* Watch the logs on the secondaries:

``sh 
$ tail -f /var/log/postgresql/repmgrd.log
[2018-12-21 22:06:04] [WARNING] unable to connect to upstream node ""node2.raveland.priv"" (node ID: 3)
[2018-12-21 22:06:04] [INFO] checking state of node 3, 1 of 6 attempts
...
[2018-12-21 22:06:54] [NOTICE] this node is the winner, will now promote itself and inform other nodes
[2018-12-21 22:06:54] [INFO] promote_command is:
  "/usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file"
[2018-12-21 22:06:54] [NOTICE] redirecting logging output to "/var/log/postgresql/repmgrd.log"
[2018-12-21 22:06:54] [NOTICE] promoting standby to primary
[2018-12-21 22:06:54] [DETAIL] promoting server ""node0.raveland.priv"" (ID: 1) using "sudo /usr/bin/pg_ctlcluster 10 main promote"
[2018-12-21 22:06:54] [DETAIL] waiting up to 15 seconds (parameter "promote_check_timeout") for promotion to complete
[2018-12-21 22:06:54] [NOTICE] STANDBY PROMOTE successful
[2018-12-21 22:06:54] [DETAIL] server ""node0.raveland.priv"" (ID: 1) was successfully promoted to primary
INFO:  node 2 received notification to follow node 1
[2018-12-21 22:06:54] [INFO] switching to primary monitoring mode
```

This secondary server has been chosen as new primary.

```
[2018-12-21 22:06:54] [WARNING] unable to reconnect to node 3 after 6 attempts
[2018-12-21 22:06:54] [INFO] follower node awaiting notification from a candidate node
[2018-12-21 22:06:55] [NOTICE] redirecting logging output to "/var/log/postgresql/repmgrd.log"
[2018-12-21 22:06:55] [NOTICE] setting node 2s primary to node 1
[2018-12-21 22:06:55] [NOTICE] restarting server using "sudo /usr/bin/pg_ctlcluster 10 main restart"
[2018-12-21 22:07:03] [WARNING] unable to connect to old upstream node 3 to remove replication slot
[2018-12-21 22:07:03] [HINT] if reusing this node, you should manually remove any inactive replication slots
[2018-12-21 22:07:03] [NOTICE] STANDBY FOLLOW successful
[2018-12-21 22:07:03] [DETAIL] standby attached to upstream node ""node0.raveland.priv"" (node ID: 1)
  INFO:  set_repmgrd_pid(): provided pidfile is /tmp/repmgrd.pid
[2018-12-21 22:07:03] [NOTICE] node 2 now following new upstream node 1
[2018-12-21 22:07:03] [INFO] resuming standby monitoring mode
[2018-12-21 22:07:03] [DETAIL] following new primary ""node0.raveland.priv"" (node id: 1)
```

This one has been updated to follow the new primary.

* Verify the status

```sh
$ repmgr -f /etc/repmgr.conf cluster show
ID | Name                  | Role    | Status    | Upstream              | Location | Connection string                                                   
----+-----------------------+---------+-----------+-----------------------+----------+----------------------------------------------------------------------
1  | "node0.raveland.priv" | primary | * running |                       | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
2  | "node1.raveland.priv" | standby |   running | "node0.raveland.priv" | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
3  | "node2.raveland.priv" | primary | - failed  |                       | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2

WARNING: following issues were detected
   unable to connect to node ""node2.raveland.priv"" (ID: 3)
```

Everything is fine !

#### The old primary comes back
The problem we had with _repmgr_ is the same with _repmgrd_. The old primary **will stay** primary !!

* Start the old primary : `systemctl start postgresql`
* Verify :

```sh
$ repmgr -f /etc/repmgr.conf cluster show
ID | Name                  | Role    | Status               | Upstream              | Location | Connection string                                                   
----+-----------------------+---------+----------------------+-----------------------+----------+----------------------------------------------------------------------
1  | "node0.raveland.priv" | standby | ! running as primary | "node2.raveland.priv" | default  | host=node0.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
2  | "node1.raveland.priv" | standby |   running            | "node2.raveland.priv" | default  | host=node1.raveland.priv user=repmgr dbname=repmgr connect_timeout=2
3  | "node2.raveland.priv" | primary | * running            |                       | default  | host=node2.raveland.priv user=repmgr dbname=repmgr connect_timeout=2

WARNING: following issues were detected
   node ""node0.raveland.priv"" (ID: 1) is registered as standby but running as primary
```

The solution is the same as before:

* Stop the old primary: `systemctl stop postgresql`
* Stop repmgrd : `systemctl stop repmgrd`
* Rejoin:

```sh
$ repmgr -f /etc/repmgr.conf node service --action=stop --checkpoint
$ repmgr -f /etc/repmgr.conf -d 'host=node0.raveland.priv user=repmgr dbname=repmgr' node rejoin
```

* Start repmgrd: `systemctl start repmgrd`

## Pgbouncer joins the dance
Now that we have a PostgreSQL cluster with automation thanks to repmgrd, it could be usefull to add pgbouncer into the architecture.

Like this, we will have only one entrypoint for our applications.

### Basic configuration for pgbouncer
At the beginning of this post, we already installed pgbouncer on the machine. Now we will do a basic configuration.

* Edit the file `/etc/pgbouncer/pgbouncer.ini`

```properties
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid

listen_addr = *
listen_port = 5432

unix_socket_dir = /var/run/postgresql

auth_type = trust
auth_file = /etc/pgbouncer/userlist.txt

admin_users = postgres
stats_users = postgres

pool_mode = transaction

server_reset_query = DISCARD ALL

%include /etc/pgbouncer/pgbouncer.database.ini
```

* Create an empty file `/etc/pgbouncer/pgbouncer.database.ini`
* Edit the file `/etc/pgbouncer/userlist.txt`

```
"postgres" "MySuperPasswordAdmin"
```

* Set rights: `chown postgres:postgres /etc/pgbouncer /etc/pgbouncer/pgbouncer.database.ini`
* Restart pgbouncer: `systemctl restart pgbouncer`

### Create a database and a user in PostgreSQL
In order to test our setup, we need to create an user and a database in the PostgreSQL cluster (on the primary of course).

```sh
  $ psql
  psql (10.6 (Debian 10.6-1.pgdg90+1))
  Type "help" for help.
  postgres=# create role pea login ;
  CREATE ROLE
  postgres=# \password pea
  Enter new password:
  Enter it again:
  postgres=# create database pea owner pea ;
  CREATE DATABASE
```

### Adapt first configuration for pgbouncer
Now that we have a user, a database and a primary server, we will finish the first configuration for pgbouncer.

* Edit the file /etc/pgbouncer/userlist.ini and setup your username and password

```
  "pea" "MySuperPassword"
```

* Edit the file `/etc/pgbouncer/pgbouncer.database.ini`

```
  [databases]
  pea= host=node1.raveland.priv
```

* Test pgbouncer (directly from the pgbouncer machine)
  
```sh  
  psql -U pea
  psql (10.6 (Debian 10.6-1.pgdg90+1))
  Type "help" for help.

  pea=>
```

Everything works !

### Integrate pgbouncer with repmgr
The purpose is to have something automatic and each time the cluster’s topology is modified, pgbouncer must be notified.

As you may have understood, when the topology will change, we will generate a new file called `pgbouncer.database.ini` and send it to pgbouncer.

For this, we need to adjust the configuration of _repmgr_.

But let me explain you how we will make this happen:

* All our PostgreSQL nodes are up and running. Pgbouncer is happy with it’s configuration pointing to the actual primary.
* The primary fails…
* repmgrd will choose a new primary
* The promote command will be executed on the new primary. By default, it’s `/usr/bin/pg_ctlcluster 10 main promote`. So we will change this command and like this we will generate a new file with the new primary info, do the promote and send the file to pgbouncer.
* Once the promote finished, the other secondaries will execute a _follow_

### Update repmgr configuration
The first thing to do is to write a small script that will :

* Do the promote
* Generate the new configuration file for pgbouncer
* Send it to pgbouncer
* Create a file `/usr/local/bin/promote.sh`

```sh
#!/usr/bin/env bash
set -e
set -u

PGBOUNCER_DATABASE_INI_NEW="/tmp/pgbouncer.database.ini"
PGBOUNCER_HOSTS="pgbouncer.raveland.priv"
DATABASES="pea"

# Pause pgbouncer
for h in ${PGBOUNCER_HOSTS}
do
  for d in ${DATABASES}
  do
      psql -U postgres -h ${h} -p 5432 pgbouncer -tc "pause ${d}"
  done
done

# Promote server
sudo /usr/bin/pg_ctlcluster 10 main promote

# Generate new config file for pgbouncer
echo -e "[databases]\n" > ${PGBOUNCER_DATABASE_INI_NEW}
for d in ${DATABASES}
do
  echo -e "${d}= host=$(hostname -f)\n" >> ${PGBOUNCER_DATABASE_INI_NEW}
done

# Copy new config file, reload and resume pgbouncer
for h in ${PGBOUNCER_HOSTS}
do
  for d in ${DATABASES}
  do
      rsync -a ${PGBOUNCER_DATABASE_INI_NEW} ${h}:/etc/pgbouncer/pgbouncer.database.ini
      psql -U postgres -h ${h} -p 5432 pgbouncer -tc "reload"
      psql -U postgres -h ${h} -p 5432 pgbouncer -tc "resume ${d}"
  done
done

rm ${PGBOUNCER_DATABASE_INI_NEW}
```

Please note that you have to fill the *PGBOUNCER_HOSTS* and _DATABASES_ variables.

* Be sure that _postgres_ user can execute this file : `chmod 755 /usr/local/bin/promote.sh`
* Update `/etc/repmgr.conf` on all PostgreSQL machines and change this line

```
service_promote_command = '/usr/local/bin/promote.sh'
```

### Let’s try
At this point, you should have the script `/usr/local/bin/promote.sh` deployed on all your PostgreSQL machines and the file /etc/repmgr.conf updated with the latest change.

* Shutdown the primary: `systemctl stop postgresql`
* Wait for the new primary
* Watch the pgbouncer logs:

```sh
$ tail -f /var/log/postgresql/pgbouncer.log
2018-12-21 23:46:51.141 6182 LOG C-0x563b36d38e50: pgbouncer/postgres@192.168.1.51:56442 login attempt: db=pgbouncer user=postgres tls=no
2018-12-21 23:46:51.141 6182 LOG PAUSE 'pea' command issued
2018-12-21 23:46:51.142 6182 LOG C-0x563b36d38e50: pgbouncer/postgres@192.168.1.51:56442 closing because: client close request (age=0)
2018-12-21 23:46:51.566 6182 LOG C-0x563b36d38e50: pgbouncer/postgres@192.168.1.51:56446 login attempt: db=pgbouncer user=postgres tls=no
2018-12-21 23:46:51.566 6182 LOG RELOAD command issued
2018-12-21 23:46:51.566 6182 LOG C-0x563b36d38e50: pgbouncer/postgres@192.168.1.51:56446 closing because: client close request (age=0)
2018-12-21 23:46:51.605 6182 LOG C-0x563b36d38e50: pgbouncer/postgres@192.168.1.51:56448 login attempt: db=pgbouncer user=postgres tls=no
2018-12-21 23:46:51.605 6182 LOG RESUME 'pea' command issued
```

Victory !!!

## Conclusion
The purpose of this (long) post was to show you how to setup a PostgreSQL cluster with _repmgr_ and _pgbouncer_.

Of course, I can’t detail all the options of : and you have to do your homeworks :)

But now, at least, you have the basis.

Enjoy :wink:

[1]: https://repmgr.org/docs/4.2/index.html
[2]: https://repmgr.org/docs/repmgr.html#PERFORMING-SWITCHOVER
[3]: https://repmgr.org/docs/repmgr.html#REPMGR-CLUSTER-MATRIX
[4]: https://repmgr.org/docs/repmgr.html#REPMGR-CLUSTER-CROSSCHECK
[5]: https://raw.githubusercontent.com/2ndQuadrant/repmgr/master/repmgr.conf.sample

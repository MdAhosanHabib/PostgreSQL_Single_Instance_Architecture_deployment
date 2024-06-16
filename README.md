# PostgreSQL 16.3 Single Instance Architecture & Deployment Redhat Linux 9

<img width="387" alt="PostgreSQL architecture photo" src="https://github.com/MdAhosanHabib/PostgreSQL_Single_Instance_Architecture_deployment/assets/43145662/ffd4bd9c-f05e-477f-b077-3d80fbf255db">


## 1 Client Interaction
#### Client Application:
The client application makes database requests and receives results via an API. This can be any application or service that needs to interact with the PostgreSQL database.

#### Client Interface Library:
The client interface library handles communication between the client application and the PostgreSQL server. This library could be a part of the client application or a separate component.


## 2 Server Processes
#### Postmaster Daemon Process:
The postmaster daemon process is the main PostgreSQL process responsible for initializing connections, handling authentication and authorization, and managing other server processes.
When a client request is received, the postmaster forks a new backend process (postgres backend process) to handle the individual connection.

#### Postgres Backend Process:
Each client connection is managed by a separate backend process (postgres). This process executes SQL queries and returns results to the client.


## 3 System Memory
#### Per Backend Memory:
Each backend process has its own memory space for handling operations like sorting, hashing, and temporary data storage. Key components include:

work_mem

temp_buffer

maintenance_work_mem

catalog_cache

optimizer/executor


#### PostgreSQL Shared Memory:
Shared memory is used for common tasks that need to be accessed by multiple backend processes. Key buffers and caches include:

Shared Buffers: Used for caching data pages.

WAL Buffers: Temporary storage for WAL (Write-Ahead Logging) records before they are written to disk.

Temp Buffers: Used for temporary tables and intermediate query results.

CLOG Buffers: Used for transaction commit logs.

Other Buffers: Various other internal buffers.


## 4 PostgreSQL Instance Components
Utility Processes: Several utility processes handle background tasks and ensure smooth operation of the PostgreSQL instance. These include:

BGWriter (Background Writer): Writes dirty pages from shared buffers to disk.

WAL Writer: Writes WAL records from WAL buffers to WAL files.

Auto Vacuum: Reclaims storage by removing obsolete data and updating statistics.

Stats Collector: Collects and reports database activity statistics.

Sys Logger: Logs system messages.

Archiver: Archives WAL files for backup and replication.

Checkpointer: Ensures that data changes are regularly written to disk to facilitate crash recovery.

Logical Replication Launcher: Handles logical replication tasks.

## 5 Physical Files
Data Files:
Data files store the actual database data on disk.

WAL Files:
WAL (Write-Ahead Logging) files record all changes made to the data for crash recovery and replication.

Archive Files:
Archived WAL files are stored for long-term retention and disaster recovery.

Log Files:
Log files record various system and error messages for debugging and auditing.

## 6 Data Flow
When a client application sends a request, the postmaster daemon process handles the connection setup, authentication, and authorization.

A new backend process (postgres) is forked to handle the client’s SQL queries.

The backend process interacts with the shared memory and performs necessary operations, utilizing various buffers and caches.

Utility processes ensure smooth operation by handling background tasks like writing data to disk, archiving, vacuuming, and logging.

Data changes are written to WAL files, ensuring durability and enabling crash recovery. These changes are periodically flushed to data files by the checkpointer.

Archived WAL files are stored for backup and recovery purposes.


# Now step down the deployment of PostgreSQL 16.3 in Linux 9
## Installation and Super user creation
```bash
-- download the required files
https://download.postgresql.org/pub/repos/yum/16/redhat/rhel-9.3-x86_64/


-- go to this dir
cd /home/team


-- upload the required packages
postgresql16-16.3-1PGDG.rhel9.x86_64.rpm
postgresql16-contrib-16.3-1PGDG.rhel9.x86_64.rpm
postgresql16-libs-16.3-1PGDG.rhel9.x86_64.rpm
postgresql16-server-16.3-1PGDG.rhel9.x86_64.rpm


-- install the dependencies
dnf install lz4 -y
dnf install libperl.so.5.32 -y


-- install the packages
rpm -ivh postgresql16-libs-16.3-1PGDG.rhel9.x86_64.rpm
rpm -ivh postgresql16-16.3-1PGDG.rhel9.x86_64.rpm
rpm -ivh postgresql16-server-16.3-1PGDG.rhel9.x86_64.rpm
rpm -ivh postgresql16-contrib-16.3-1PGDG.rhel9.x86_64.rpm


-- check the Linux utilities
ll /usr/pgsql-16/bin/


-- create custom data dir and give permission
mkdir -p /data/postgresql163/datadir
chown -R postgres:postgres /data/postgresql163

mkdir -p /data/postgresql163/wal_archive
chown -R postgres:postgres /data/postgresql163/wal_archive


-- initialize the database
sudo -i -u postgres
/usr/pgsql-16/bin/initdb -D /data/postgresql163/datadir
exit


-- add the custom datadir to Linux service
vi /usr/lib/systemd/system/postgresql-16.service
Environment=PGDATA=/data/postgresql163/datadir


-- reload and start the postgresql
systemctl daemon-reload
systemctl start postgresql-16
systemctl enable postgresql-16
systemctl status postgresql-16


-- check the Default datadir after install
sudo -u postgres psql

SHOW data_directory;
exit


-- set pass for super user
sudo -u postgres psql
ALTER USER postgres WITH PASSWORD 'pGTest';

vi /data/postgresql163/datadir/pg_hba.conf
# "local" is for Unix domain socket connections only
local   all             all                                     md5

systemctl reload postgresql-16


-- database log
tail -1000f /data/postgresql163/datadir/log/postgresql-Tue.log
```

## Configure the Parameters of PostgreSQL instance
```bash
-- Configuration file for 8GB RAM
vi /data/postgresql163/datadir/postgresql.conf
# Memory settings
shared_buffers = 2GB
effective_cache_size = 6GB
maintenance_work_mem = 256MB
work_mem = 32MB

# wal file settings
wal_buffers = -1 # -1 sets based on shared_buffers
min_wal_size = 512MB
max_wal_size = 1GB

# Checkpoints settings
checkpoint_completion_target = 0.7

# Connection settings
max_connections = 200

# archive settings for wal
archive_mode = on
archive_command = 'test ! -f /data/postgresql163/wal_archive/%f && cp %p /data/postgresql163/wal_archive/%f'

# Connection settings
listen_addresses = '*'


-- network allowing
vi /data/postgresql163/datadir/pg_hba.conf
host    all             all             0.0.0.0/00              md5


systemctl status postgresql-16
systemctl restart postgresql-16
```

## User Schema Tablespace management best practices
```bash
-- create dir & tablespaces
mkdir -p /data/postgresql163/tablespaces/testpay
chown -R postgres:postgres /data/postgresql163/tablespaces/testpay

sudo -u postgres psql
CREATE TABLESPACE testpaytbs LOCATION '/data/postgresql163/tablespaces/testpay';


-- create database
CREATE DATABASE testpaydb
  WITH OWNER = postgres
       TEMPLATE = template0
       ENCODING = 'UTF8'
       LC_COLLATE = 'en_US.UTF-8'
       LC_CTYPE = 'en_US.UTF-8'
       TABLESPACE = testpaytbs;


-- use testpaydb
sudo -u postgres psql
\c testpaydb

SELECT schema_name 
FROM information_schema.schemata 
WHERE schema_name NOT LIKE 'pg_%' 
AND schema_name != 'information_schema';


-- create schema in the database
CREATE SCHEMA ahtestpay AUTHORIZATION postgres;


-- user create and give permission
CREATE USER testpay WITH PASSWORD 'testpay10';
GRANT CONNECT ON DATABASE testpaydb TO testpay;
GRANT USAGE ON SCHEMA ahtestpay TO testpay;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA ahtestpay TO testpay;
ALTER DEFAULT PRIVILEGES IN SCHEMA ahtestpay GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO testpay;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA ahtestpay TO testpay;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA ahtestpay TO testpay;
GRANT CREATE ON SCHEMA ahtestpay TO testpay;


-- set default schema ahtestpay for testpay user
psql -U postgres -d testpaydb
ALTER USER testpay SET search_path TO ahtestpay;
SELECT usename, useconfig FROM pg_user WHERE usename = 'testpay';


-- login with user
psql -U testpay -W -h 10.0.10.10 -d testpaydb
pass: testpay10

SELECT schema_name 
FROM information_schema.schemata 
WHERE schema_name NOT LIKE 'pg_%' 
AND schema_name != 'information_schema';


-- create table and insert data
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    age INTEGER
);

INSERT INTO employees (name, age) VALUES ('John Doe', 30);
INSERT INTO employees (name, age) VALUES ('Jane Smith', 25);

testpaydb=> select * from employees;
```

## PostgreSQL Credentials
```bash
-- super user
IP: 10.0.10.10
Port: 5432
User: postgres
Pass: pGTest

-- app user
IP: 10.0.10.10
Port: 5432
User: testpay
Pass: testpay10
Database: testpaydb
Schema: ahtestpay
```

#### That's all
### Regards Ahosan

Medium: https://medium.com/@ahosanhabib.974/postgresql-16-3-single-instance-architecture-deployment-redhat-linux-9-3ee3ec0905cd

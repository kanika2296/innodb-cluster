# Creating innodb-cluster using Docker containers

## Setting up docker
Install Docker Toolbox (Windows)

## Create docker Network
``` docker network create innodbnet```
## Setup Containers with MySQL server 8.0
```
for N in 1 2 3 
  do docker run -d --name=mysql$N --hostname=mysql$N --net=innodbnet \
      -e MYSQL_ROOT_PASSWORD=root mysql/mysql-server:8.0
done
```
Ensure that the status of the containers is "healthy" by running ```docker ps -a```. If it is not "healthy" wait for a few minutes and re-run.

## Grant new user access
Create a user named inno with password  inno and grant it the required access in each of the MySQL servers.
```
for N in 1 2 3 
do docker exec -it mysql$N mysql -uroot -proot \
  -e "CREATE USER 'inno'@'%' IDENTIFIED BY 'inno';" \
  -e "GRANT ALL privileges ON *.* TO 'inno'@'%' with grant option;" \
  -e "reset master;"
done
```
Check whether the users were created successfuly:
```
for N in 1 2 3 
do docker exec -it mysql$N mysql -uinno -pinno \
  -e "SHOW VARIABLES WHERE Variable_name = 'hostname';" \
  -e "SELECT user FROM mysql.user where user = 'inno';"
done
```
# Configure the MySQL servers into a InnoDB Cluster
Run this command to access 'mysql1'
```docker exec -it mysql1 mysqlsh -uroot -proot -S/var/run/mysqld/mysqlx.sock```
## Verfiy is the instance is ready for Innodb Cluster
Run the command and type 'inno' when promoted for password
```dba.checkInstanceConfiguration("inno@mysql1:3306")```

Repeat this step on all the instances

## Configure MySQL Servers for InnoDB Cluster
Run ```dba.configureInstance("inno@mysql1:3306")```
Type 'Y' for both the questions. The instance will restart.
If it doesnot start automatically run ```docker restart mysql1 ```.
Repat this step for remaing 2 instances and run ```docker restart mysql2 mysql3```

# Creating Innodb Cluster
Run 
```
docker exec -it mysql1 mysqlsh -uroot -proot -S/var/run/mysqld/mysqlx.sock
```
Connect to the seed (primary) instance and type the password inno:
```\c inno@mysql1:3306```

## Create Cluster 
```var cluster = dba.createCluster("mycluster")```
Run the command ```cluster.status()``` to check cluster status and ```cluster.describe()``` for more detials.

## Adding instances to cluster
To add instance run ```cluster.addInstance("inno@mysql2:3306")```
Choose option 'Incremental' by typing 'I'.

Now check the status of cluster by running ```cluster.status()```
```
 "status": "OK_NO_TOLERANCE", 
````
Innodb cluster requires minimum of three instance to be fault tolerant

To add another instance run ```cluster.addInstance("inno@mysql3:3306")```
Now check the status of cluster by running ```cluster.status()```
```
 "status": "OK", 
````
Exit Shell.

# Configuring MySQL Router

Run the following commands
```
docker run -d --name mysql-router --net=innodbnet \
   -e MYSQL_HOST=mysql1 \
   -e MYSQL_PORT=3306 \
   -e MYSQL_USER=inno \
   -e MYSQL_PASSWORD=inno \
   -e MYSQL_INNODB_CLUSTER_MEMBERS=3 \
   mysql/mysql-router
   
  docker logs mysql-router
 ```
  Succesfull execution should display Successfully contacted .....## MySQL Classic protocol

- Read/Write Connections: localhost:6446
- Read/Only Connections:  localhost:6447

# Adding data
Create a new MySQL 8.0 container named mysql-client which will server as a client:
```
$ docker run -d --name=mysql-client --hostname=mysql-client --net=innodbnet \
   -e MYSQL_ROOT_PASSWORD=root mysql/mysql-server:8.0
```
Add some data to the cluster. Point to remember mysql-client uses port : 6446 R/W to be able to write data to seed instance
```
$ docker exec -it mysql-client mysql -h mysql-router -P 6446 -uinno -pinno \
  -e "create database TEST; use TEST; CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY) ENGINE=InnoDB; show tables;" \
  -e "INSERT INTO TEST.t1 VALUES(1); INSERT INTO TEST.t1 VALUES(2); INSERT INTO TEST.t1 VALUES(3);"
```
# Verifying Replication
To verify replication, login to MySQL as R/O(read-only) instance :

Add new data using R/W port 6446 , run :
```
$ docker exec -it mysql-client mysql -h mysql-router -P 6446 -uinno -pinno \
  -e "use TEST;" \
  -e "INSERT INTO TEST.t1 VALUES(99);"

```
```
docker exec -it mysql-client mysql -h mysql-router -P 6447 -uinno -pinno \
  -e "SELECT * FROM TEST.t1;"
  ```
# Fault Tolerance
To check which instance is running as Primary , run :
```
docker exec -it mysql-client mysqlsh -h mysql-router -P 6447 -uinno -pinno
\sql
SELECT MEMBER_HOST, MEMBER_ROLE FROM performance_schema.replication_group_members;
\exit
```
To stop primary instance , run:
```docker stop primary_name```

What will happen now when primary node is down?
Run :
```
docker exec -it mysql-client mysqlsh -h mysql-router -P 6447 -uinno -pinno
var cluster = dba.getCluster("mycluster")
cluster.status()
```
The cluster is still up and running, but we will have a new primary, say mysql2. The node mysql1 (seed) is now with status "(MISSING)" and mode "n/a".

Restart the instance , run :
```
docker start primary_name
```
Run :
```
cluster.status()
```
The status of instance (primary) will change from RECOVERING - > ONLINE. Now this instance will be a R/O instance.

# Switching between Primary and Multi-Primary Mode
Primary mode is a single master (R/W) mode, to switch , run:
```
docker exec -it mysql-client mysqlsh -h mysql-router -P 6447 -uinno -pinno
var cluster = dba.getCluster("mycluster")
cluster.switchToMultiPrimaryMode()
```
Run :
```
cluster.status()
```
To verify check if all instances are running in "R/W" mode.

To switch back to primary , run :
```
cluster.switchToSinglePrimaryMode("inno@mysql1:3306");
```

# Scaling Up
When adding a new instance, a node has to be provisioned first before it's allowed to participate with the replication group. The provisioning process will be handled automatically by MySQL. Also, you can check the instance state first whether the node is valid to join the cluster by using checkInstanceState() function as previously explained.

To add a new DB node, use the addInstances() function and specify the host:
```
cluster.addInstance("inno@mysql4:3306")
````
To verify the new cluster size, Run :
```
cluster.status()
```

# Scaling Down
To remove a node, connect to any of the DB nodes except the one that we are going to remove and use the removeInstance() function with the database instance name:
```
cluster.removeInstance("inno@mysql4:3306")
```

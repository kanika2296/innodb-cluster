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
 "status": "OK", 
  "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
````
Innodb cluster requires minimum of three instance to be fault tolerant

To add another instance run ```cluster.addInstance("inno@mysql3:3306")```


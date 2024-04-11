# Migrating-to-the-Cloud-with-Conterization_P20
## Migrate a tooling and PHP-TODO app into a containerized application
# Install Docker and prepare for migration to the Cloud
 First we need to install docker engine which is a client-server application that contains 
 - A server with a long-running daemon process `dockerd`.
- APIs that specify interfaces that programs can use to talk to and instruct the Docker daemon.
- A command-line interface (CLI) client docker
## For tooling app
-------------
Let us start assembling our application from the Database layer- we will use a pre-built MySQL database container, configure it, and make sure it is ready to receive requests from our PHP application.
### Step1: Pull MYSQL Docker image from [Docker Hub registry](https://hub.docker.com/)
Start by pulling the appropriate Docker image for MySQL. You can download a specific version or opt for the latest release, as seen in the following command:
`docker pull mysql/mysql-server:latest` 
![sqlpull](images/sqlpull.png)
List the images to check that you have downloaded them successfully:
`docker images ls`
![sql-image](images/imagesql.png)

### Step 2: Create a network and run MySQL
Creating a custom network isn't always necessary because Docker automatically assigns containers to the default network, typically using the Bridge driver. You can confirm this by running docker network ls.

However, there are scenarios where a custom network is useful. For instance, if you need to control the IP address range of containers in your application stack, creating a network with a specified --subnet becomes essential.

In our case, for clarity and organization, we'll create a dedicated network with its own subnet. This will ensure seamless connectivity between MySQL and our application.

`docker network create --subnet=172.18.0.0/24 tooling_app_network`

let us create an environment variable to store the MySQL root password 
`export MYSQL_PW=olalekan`
 lets run the mysql using our already created network 
 ` docker run --network tooling_app_network -h mysqlserverhost --name=mysql-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW -d mysql/mysql-server:latest`

Flags used

-d runs the container in detached mode
--network connects a container to a network
-h specifies a hostname
If the image is not found locally, it will be downloaded from the registry but we already pulled the MySQL image in step 1.

Verify the container is running
`docker ps`

![dockerps](images/dockerps.png)
### Step 4. Create database

As you already know, it is best practice not to connect to the MySQL server remotely using the root user. Therefore, we will create an SQL script that will create a user we can use to connect remotely.

Create a file and name it create_user.sql and add the below code in the file:

`CREATE USER '<user>'@'%' IDENTIFIED BY '<client-secret-password>';
GRANT ALL PRIVILEGES ON * . * TO '<user>'@'%';`

![create-user](images/create-user.png)

Run the script:
docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < ./create_user.sql

### Step 3: Connecting to MySQL Server using another MySQL client container

By leveraging another MySQL client container to connect to the MySQL server, we eliminate the need to install any client tools on our local machine. This approach allows us to interact with the MySQL server seamlessly without directly connecting to the container running the server. As a result, managing databases becomes more efficient and convenient.

Run the MySQL Client Container
`docker run --network tooling_app_network --name mysql-client -it --rm mysql mysql -h myserverhost -u "<user-created-from-the-sql-script>" -p`

![show-database](images/showdb.png)



Flags used:
- --name gives the container a name
- -it runs in interactive mode and Allocate a pseudo-TTY
- --rm automatically removes the container when it exits
- 	--network connects a container to a network
-	-h a MySQL flag specifying the MySQL server Container hostname
-	-u user created from the SQL script
-	-p password specified for the user created from the SQL script


**Prepare database schema**

Now you need to prepare a database schema so that the Tooling application can connect to it.
1.	Clone the Tooling-app repository from here 
git clone this [repo](https://github.com/babslekson/tooling.git)
2.	On your terminal, export the location of the SQL file
export tooling_db_schema=~/tooling/html/tooling_db_schema.sql
You can find the tooling_db_schema.sql in the html folder of cloned repo.
3.	Use the SQL script to create the database and prepare the schema. With the docker exec command, you can execute a command in a running container.
docker exec -i mysql-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema
4.	Update the db_conn.php file with connection details to the database
```bash
$servername = "mysqlserverhost";
$username = "<user>";
$password = "<client-secret-password>";
$dbname = "toolingdb";
```



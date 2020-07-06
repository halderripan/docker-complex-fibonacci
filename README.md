# docker-complex-fibonacci

## A multi-container Docker deployment

A multicontainer fibonacci application - React, Redis, Postgres, Nginx

## Development Process Diagram
![Development Process Diagram](/assets/Dev_Process_Diagram.png "Dev Process Diagram.png")

## Production Process Diagram
![Production Process Diagram](/assets/Prod_Process_Diagram.png "Prod Process Diagram.png")

### Step 1. All Configurations

#### Docker config

We want 4 project apps:

* client
* server
* worker
* nginx

as 4 containers.

There are 4 ```Dockerfile.dev``` files in each of the app folders for development servers.

Build each app in dev env to test it:

    docker build -f Dockerfile.dev .
    docker run ID

Either way there is a ```docker-compose.yml``` file in the project root DIR to build and run all the 4 dev files together.

    docker-compose up --build

There are 4 ```Dockerfile``` files in each of the app folders for production servers.


##### Environment variables

Setup environment variables with docker.

Environment variables are created in the runtime, not stored in an docker image.

* Travis-CI Env Vars:
    * DOCKER_ID
    * DOCKER_PASSWORD
    * AWS_ACCESS_KEY_ID -> User having EB deploy access
    * AWS_SECRET_ACCESS_KEY

* AWS EB Env Vars:
    * REDIS_HOST -> Dev : Redis docker image :: Prod : AWS ElastiCache (Open ElastiCache in a new tab:Redis:Description:Primary Endpoint, use this (without a port))
    * REDIS_PORT -> 6379
    * PGUSER -> use values set in [RDS section](#rds)
    * PGHOST -> Dev : Postgres docker image :: Prod : AWS RDS Postgres DB (Open RDS AWS in a new tab:Connectivity:Endpoint, use this)
    * PGDATABASE -> use values set in [RDS section](#rds)
    * PGPASSWORD -> use values set in [RDS section](#rds)
    * PGPORT -> 5432

#### Port mapping

A ```default.conf``` which adds configuration rules to Nginx.

#### Nginx Path Routing Diagram
![Nginx Path Routing Diagram](/assets/Nginx_Path_Routing_Diagram.png "Nginx Path Routing Diagram.png")

#### Nginx Configuration File Flow
![Nginx Configuration File Flow](/assets/Nginx_Configuration_File_Flow.png "Nginx Configuration File Flow.png")

#### Websocket connection

Allow WebSocket connections in nginx conf for Dev


### Step 2. Test and deploy with Travis-CI and Elastic Beanstalk AWS

#### Multi Container Set Up Flow Diagram
![Multi Container Set Up Flow Diagram](/assets/Multi_Container_Set_Up_Flow_Diagram.png "Multi Container Set Up Flow Diagram.png")

#### Travis-CI

A ```.travis.yml``` file which has the following:

- ```services``` : docker 
- ```before_install``` : build the docker client application
- ```script```: Run tests
- ```after_success```: Build and Push 4 docker images to my docker hub repos.
- ```deploy```: Upload the code to AWS EB's S3 bucket and eploy them to AWS EB.


#### Elastic Beanstalk AWS

A new AWS Elastic Beanstalk instance via AWS dashboard with single container deployment (choose ```multi-docker``` in settings.)

Use ```Dockerrun.aws.json```.

Pull the docker hub images with Elastic Beanstalk AWS in ```Dockerrun.aws.json```.


```Dockerrun.aws.json``` stores container/tasks [definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definitions) - instructions on how to run a single container (instance of an image).

###### VPC - Virtual Private Cloud

You get it (one default) in each of very specific different regions in EB AWS.

In dev env we had Redis and Postgres containers.
In prod env we are going to use:

* AWS ElastiCache instead of Redis
* AWS Relational Database Service (RDS)

in Elastic Beanstalk (EB) instance.

###### Why AWS ElastiCache and AWS RDS

* Super easy to scale
* Built in logging + maintenance
* Probably better security than what we can do
* Easier to migrate off of EB with

###### Why AWS ElastiCache

* Automatically creates and maintains Redis instances for you, you dont have to be a Redis specialist

###### Why AWS RDS

* Automatically creates and maintains Postgres instances for you
* Automated backups and rollbacks

###### RDS

Configure in the same region where EB has been created.

Create database in Amazon RDS, type PostgreSQL.
Fill in:

* DB instance identifier
* Master username
* Master password

Key names need to be equal to your ```docker-compose.yml``` ```api.environment``` config.
Values can be different than set up in ```docker-compose.yml```.

###### ElastiCache

Create Redis database cluster in Amazon ElastiCache (Cluster Mode disabled).
Change Node type to t2 cache.t2.micro (low performance, the cheapest) and Number of replicas to 0. Check subnets. Save with default VPC region number.

###### Security Group

To get services (EB, RDS, ElastiCache) connect with each other we need to create a security group (firewall rules) in AWS, which is: allow any traffic from any other AWS service that has this security group.

Create a new Security Group from VPC service dashboard. Save with default VPC region number. Edit Inbound Rule for this group. Rule type: Custom TCP Rule, Port Range 5432-6379, Source Custom - just created security Group ID.

Now apply the security group ID to each of 3 services: EB, RDS, ElastiCache:

* In Redis AWS:Modify:VPC Security Groups.
* In Amazon RDS:AWS Modify:Network & Security:Security group:Apply immediately.
* In Elastic Beanstalk Service:Configuration:instances:Modify:EC2 security groups

#### IAM AWS keys for deployment

The only file that counts here to be send ver to EB is ```docker-composer.yml```.

Specify deploy section in ```.travis.yml``` with

* ```provider: elasticbeanstalk```
* region - you will find region in your aws domain subdomain
* app - it is the Elastic Beanstalk instance name.
* env - it is the Elastic Beanstalk instance env name, ends with ```-env```
* bucket_name - it is Amazon S3 bucket, go to AWS S3, find bucker name
* bucket_path - same as app

Generate ```$AWS_ACCESS_KEY``` and ```AWS_SECRET_KEY``` on IAM AWS dashboard:Users:Add User:Check Programmatic access:Permissions:Attach existing policies directly:Filter beanstalk:You can check everything or Provides full access to AWS Elastic Beanstalk:Create user without a permissions boundary:No tags:Create User

Copy ```Access key ID``` and ```Secret access key```.

##### Update ```.travis.yml``` file to deploy over to EB:

Generate ```access_key_id``` and ```secret_access_key``` environment variables on Travis-CI dashboard:[project]:More options:Settings:Environment Variables

Create ```AWS_ACCESS_KEY``` key with ```Access key ID``` value.

Create ```AWS_SECRET_KEY``` key with ```Secret access key``` value.

DO NOT DISPLAY VALUE IN BUILD LOG.

#### If health status it not OK

* Check logs in EB

#### Re-deploying application

#### Cleaning Up AWS Resources

to not get charged.

We need to clean:

* Elastic Beanstalk - AWS Elastic Beanstalk dashboard:[project]:Actions:Delete application
* ElastiCache - AWS ElastiCache dashboard:Redis:[project]:Delete
* RDS - AWS RDS dashboard:[project]:Modify:Deletion protection:Uncheck, then AWS RDS dashboard:[project]:Actions:Delete

***

***

***

* Based on [udemy course](https://www.udemy.com/docker-and-kubernetes-the-complete-guide/).
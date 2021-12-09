# CONTAINERIZATION WITH DOCKER

## 1. TOOLING APP CONTAINERIZATION

Repo: https://github.com/brpo01/docker-tooling-webapp

### Docker Installation

You can learn how to install Docker Engine on your PC [here](https://docs.docker.com/engine/install/)

#### MySQL in container
Let us start assembling our application from the Database layer – we will use a pre-built MySQL database container, configure it, and make sure it is ready to receive requests from our PHP application.

**Step 1: Pull MySQL Docker Image from Docker Hub Registry**
- Start by pulling the appropriate Docker image for MySQL. You can download a specific version or opt for the latest release, as seen in the following command:

```
docker pull mysql/mysql-server:latest
```

If you are interested in a particular version of MySQL, replace latest with the version number. Visit Docker Hub to check other tags

- List the images to check that you have downloaded them successfully:

```
docker image ls
```
![2](https://user-images.githubusercontent.com/47898882/145481834-19b17180-e0aa-42c8-b790-09a443c27db5.JPG)

**Step 2: Deploy the MySQL Container to your Docker Engine**

- Once you have the image, move on to deploying a new MySQL container with:
```
docker run --name <container_name> -e MYSQL_ROOT_PASSWORD=<my-secret-pw> -d mysql/mysql-server:latest
```
Replace `<container_name>` with the name of your choice. If you do not provide a name, Docker will generate a random one

The ***-d*** option instructs Docker to run the container as a service in the background

Replace `<my-secret-pw>` with your chosen password
  
In the command above, we used the latest version tag. This tag may differ according to the image you downloaded

- Then, check to see if the MySQL container is running:

  ```
  docker ps 
  ```

  ![3](https://user-images.githubusercontent.com/47898882/145481837-b08b7fc3-eb96-4928-a268-5aa31ff0c03c.JPG)
  
You should see the newly created container listed in the output. It includes container details, one being the status of this virtual environment. The status changes from health: starting to healthy, once the setup is complete.

**Step 3: Connecting to the MySQL Docker Container**

- We can either connect directly to the container running the MySQL server or use a second container as a MySQL client. Let us see what the first option looks like.

***Approach 1***

   - Connecting directly to the container running the MySQL server:
```
 docker exec -it <DB container name or ID> mysql -uroot -p 
```
![4](https://user-images.githubusercontent.com/47898882/145481842-af75cef5-ed35-4bf8-a536-77f247d38162.JPG)

   - Provide the root password when prompted. With that, you have connected the MySQL client to the server
   - Finally, change the server root password to protect your database

***Approach 2***

   - First, create a network:

```
docker network create --subnet=172.18.0.0/24 tooling_app_network 
```

![5](https://user-images.githubusercontent.com/47898882/145481846-9eea3cae-cd17-435a-b7f3-c8a00eeb0eda.JPG)

Creating a custom network is not necessary because even if we do not create a network, Docker will use the default network for all the containers you run. By default, the network we created above is of DRIVER Bridge. So, also, it is the default network. You can verify this by running the docker network ls command.

But there are use cases where this is necessary. For example, if there is a requirement to control the cidr range of the containers running the entire application stack. This will be an ideal situation to create a network and specify the --subnet

For clarity’s sake, we will create a network with a subnet dedicated for our project and use it for both MySQL and the application so that they can connect.

- Run the MySQL Server container using the created network. But first, let us create an environment variable to store the root password:
```
export MYSQL_PW=<type password here>
```

![1](https://user-images.githubusercontent.com/47898882/145481826-c621cf92-af8b-4505-8ca0-5ecd99cb4d23.JPG)

- Then, pull the image and run the container, all in one command like below:

```
docker run --network tooling_app_network -h mysqlserverhost --name DB-server -e MYSQL_ROOT_PASSWORD=$MYSQL_PW  -d mysql/mysql-server:latest
```

- Verify the container is running:

```
docker ps -a
```

- As you already know, it is best practice not to connect to the MySQL server remotely using the root user. Therefore, we will create an SQL script that will create a user we can use to connect remotely.

Create a file and name it `create_user.sql` and add the below code in the file:
```
CREATE USER 'sql_user'@'%' IDENTIFIED BY '1234ABC';
GRANT ALL PRIVILEGES ON * . * TO 'sql_user'@'%';
```

- Run the script: 
```
docker exec -i <container name or ID> mysql -uroot -p$MYSQL_PW < ~/create_user.sql
```

![6](https://user-images.githubusercontent.com/47898882/145481847-e61a5111-9ac1-4d90-bc0b-c53145050d6e.JPG)

**Step 4: Connecting to the MySQL server from a second container running the MySQL client utility**
- Run the MySQL Client Container:

```
docker run --network tooling_app_network --name DB-client -it --rm mysql mysql -h mysqlserverhost -u mysql_user -p
```
![7](https://user-images.githubusercontent.com/47898882/145481851-02fbbedb-691f-4174-a43b-579075bf7615.JPG)

- Since it is confirmed that you can connect to your DB server from a client container, exit the mysql utility and press `Control+ C` to terminate the process thus removing the container( the container is not running in a detached mode since we didn't use **-d** flag ).

**Step 5: Prepare database schema**
Now you need to prepare a database schema so that the Tooling application can connect to it.

- Create a directory and name it tooling, then download the Tooling-app repository from github.
```
git clone https://github.com/darey-devops/tooling.git 
```

- On your terminal, export the location of the SQL file
```
export tooling_db_schema=~/environment/docker-projects/tooling/html/tooling_db_schema.sql
```

![8](https://user-images.githubusercontent.com/47898882/145481853-a57285a3-ff69-4421-b3b6-5a3d72812328.JPG)

- You can find the `tooling_db_schema.sql` in the html folder of the downloaded repository.

Use the SQL script to create the database and prepare the schema. With the docker exec command, you can execute a command in a running container.
```
docker exec -i DB-server mysql -uroot -p$MYSQL_PW < $tooling_db_schema 

```
![9](https://user-images.githubusercontent.com/47898882/145481854-8aaca160-fa2b-4b01-8026-b1f44d752d1b.JPG)

- Update the db_conn.php file with connection details to the database
`
 $servername = "mysqlserverhost";
 $username = "sql_user";
 $password = "1234ABC";
 $dbname = "toolingdb";
 `

**Step 6: Packaging, Building and Deploying the Application**
- A shell script named `start-apache` came with the downloaded repository. It will be referenced in a special file called `Dockerfile` and run with the `CMD` Dockerfile instruction. This will allow us to be able to map other ports to port 80 and publish them using **-p** in our command as we will see later on.

- Pull image from Docker registry with the code below:
```
docker pull php:7-apache-buster
```

- In the tooling directory, create a Dockerfile and paste the code below:

```
FROM php:7-apache-buster
MAINTAINER Rotimi opraise139@gmail.com

RUN docker-php-ext-install mysqli
COPY apache-config.conf /etc/apache2/sites-available/000-default.conf
COPY start-apache /usr/local/bin
RUN a2enmod rewrite

# Copy application source
COPY html /var/www
RUN chown -R www-data:www-data /var/www

CMD ["start-apache"]
```

- Ensure you are inside the folder that has the Dockerfile and build your container:
```
docker build -t tooling:0.0.1 .
```

![10](https://user-images.githubusercontent.com/47898882/145481856-1d189c68-87cb-43e0-ac29-5b61e7700606.JPG)


In the above command, we specify a parameter -t, so that the image can be tagged **tooling:0.0.1** - Also, you have to notice the **.** at the end. This is important as that tells Docker to locate the Dockerfile in the current directory you are running the command. Otherwise, you would need to specify the absolute path to the Dockerfile.

- Run the container:
```
docker run --network tooling_app_network --name website -d -h mysqlserverhost -p 8085:80 -it tooling:0.0.1
```

![13](https://user-images.githubusercontent.com/47898882/145481869-b3bc8f8d-c9f1-4651-88b4-cadf49cb83d7.JPG)

***Let us observe those flags in the command. We need to specify the --network flag so that both the Tooling app and the database can easily connect on the same virtual network we created earlier. The -p flag is used to map the container port with the host port. Within the container, apache is the webserver running and, by default, it listens on port 80. You can confirm this with the CMD ["start-apache"] section of the Dockerfile. But we cannot directly use port 80 on our host machine because it is already in use. The workaround is to use another port that is not used by the host machine. In our case, port 8085 is free, so we can map that to port 80 running in the container.***


- You can open the browser and type http://localhost:8085. The default email is test@gmail.com, the password is 12345 or you can check users' credentials stored in the toolingdb.user table.

![11](https://user-images.githubusercontent.com/47898882/145481861-0693dd26-f1ee-410b-8888-52edddfa84a4.JPG)

- Input the login credentials

![12](https://user-images.githubusercontent.com/47898882/145481866-0e606ae5-76ec-4956-933e-1f65645d277a.JPG)

### DEPLOYMENT USING DOCKER-COMPOSE

All we have done until now required quite a lot of effort to create an image and launch an application inside it. We should not have to always run Docker commands on the terminal to get our applications up and running. There are solutions that make it easy to write declarative code in YAML, and get all the applications and dependencies up and running with minimal effort by launching a single command.

We will refactor the Tooling app POC so that we can leverage the power of Docker Compose.

- First, install Docker Compose on your workstation. You can check the version of docker compose with this command: `docker-compose --version`

- Create a file and name it docker-compose.yaml
- Begin to write the Docker Compose definitions with YAML syntax. The code below represent the deployment infrastructure:

```
version: "3"

services:
  tooling_app:
    build: .
    container_name: tooling_app
    ports:
      - ${APP_PORT}:80
    volumes:
      - tooling_app:/var/www/html
    links:
      - tooling_db
    depends_on:
      - tooling_db
  tooling_db:
    image: mysql:5.7
    hostname: ${MYSQL_HOSTNAME}
    container_name: tooling_db
    restart: always
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - db:/var/lib/mysql
volumes:
  tooling_app:
  db:
```

   - Create a `.env` file to reference the variables in the tooling.yml file so they can be picked up during execution.(Make sure you have dotenv installed on your workstation). Paste the below variables in the `.env` file:

```
APP_PORT=8085
MYSQL_HOSTNAME=mysqlserverhost
MYSQL_DATABASE=toolingdb
MYSQL_USER=sql_user
MYSQL_PASSWORD=password0987654321
MYSQL_ROOT_PASSWORD=password1234567
```
- Run the command to start the containers
```
docker-compose -f tooling.yaml  up -d 
```

![14](https://user-images.githubusercontent.com/47898882/145481874-8d48301f-7f3d-4280-b3b2-b8ba70be0a19.JPG)

- Verify that the compose is in the running status:

```
docker ps  
```
![15](https://user-images.githubusercontent.com/47898882/145481878-139d3218-d185-40c6-aa16-5d84bbd5a8f7.JPG)

- Go to your browser and load http://ip-address:8085

![11](https://user-images.githubusercontent.com/47898882/145481861-0693dd26-f1ee-410b-8888-52edddfa84a4.JPG)


## 2. PHP-TODO Application Containerization using Docker

Repo: https://github.com/TheCountt/docker-php-todo

- Download or clone php-todo repository using `wget`(and unzip it) or `git clone`

![{D9F51F4D-6434-4212-860E-E12F75E0DEF4} png](https://user-images.githubusercontent.com/76074379/137538985-828c6b19-fe13-43c7-8e0c-44fb44455dc3.jpg)

![{DE7ACA34-86E0-4EBA-8F71-0E86E4B6E4D4} png](https://user-images.githubusercontent.com/76074379/137539144-d7914bad-072c-4b60-8d1e-7672221c4926.jpg)


- Write a dockerfile for php-todo app and save it in the php-todo directory
```
FROM php:7.4.24-apache
LABEL Dare=dare@zooto.io

#install zip, unzip extension, git, mysql-client
RUN apt-get update --fix-missing && apt-get install -y \
  default-mysql-client \
  git \
  unzip \
  zip \
  curl \
  wget
  
# Install docker php dependencies
RUN docker-php-ext-install pdo_mysql mysqli

# Add config files and binary file and enable webserver
COPY apache-config.conf /etc/apache2/sites-available/000-default.conf
COPY start-apache /usr/local/bin
RUN a2enmod rewrite

RUN curl -sS https://getcomposer.org/installer |php && mv composer.phar /usr/local/bin/composer

# Copy application source
COPY . /var/www
RUN chown -R www-data:www-data /var/www

EXPOSE 80

CMD ["start-apache"]
```
![{6A42FDDB-8B8D-4009-BD0F-4487E92250E7} png](https://user-images.githubusercontent.com/76074379/137537100-b2e6264f-0e10-4b97-b5f2-b34c37e5921f.jpg)

- Open the `start-apache` file, add the following commands below in addition to the commands already in the file:

```
composer install --no-plugins --no-scripts

php artisan migrate
php artisan key:generate
php artisan db:seed

apache2-foreground
```
![{8050675F-B9F2-4B9D-AA46-CA4B753FEB9A} png](https://user-images.githubusercontent.com/76074379/137628585-5dfd7524-aa8a-45b8-b7a4-35844709db1f.jpg)

- Create a docker-compose.yml in the php-todo directory and paste the code below:

```
version: "3.9"
services:
 app:
    build:
      context: .
    container_name: php-website
    network_mode: tooling_app_network
    restart: unless-stopped
    volumes:
      - app:/php-todo
    ports:
      - "${APP_PORT}:80"
    depends_on:
      - db

 db:
    image: mysql/mysql-server:latest
    container_name: php-db-server
    network_mode: tooling_app_network
    hostname: "${DB_HOSTNAME}"
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: "${DB_DATABASE}"
      MYSQL_USER: "${DB_USERNAME}"
      MYSQL_PASSWORD: "${DB_PASSWORD}"
      MYSQL_ROOT_PASSWORD: "${DB_ROOT_PASSWORD}"
    
    ports:
      - "${DB_PORT}:3306"

    volumes:
      - db:/var/lib/mysql

volumes:
  app:
  db:
  ```
![{2CB63BFB-700B-45F6-B8F7-D7CA37EC2162} png](https://user-images.githubusercontent.com/76074379/137538344-9d176fa3-86f1-4501-a776-4c92a8b884b2.jpg)


- Update the `.env` file
```
APP_PORT=8000
APP_ENV=local
APP_DEBUG=true
APP_KEY=SomeRandomString
APP_URL=http://localhost

DB_HOSTNAME=mysqlserverhost
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=sePret^1
DB_ROOT_PASSWORD=Qwerty123@
DB_PORT=3306

CACHE_DRIVER=file
SESSION_DRIVER=file
QUEUE_DRIVER=sync

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_DRIVER=smtp
MAIL_HOST=mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
```
![{935414C1-7099-4F52-9EBF-24FA2D1B85B8} png](https://user-images.githubusercontent.com/76074379/137537533-8e6079a2-2aa6-46c4-95cc-6f91b1e90290.jpg)

- Make sure you change directory to php-todo directory. Build image using this command:
```
docker build -t php-todo:latest .
```

- Make sure you change directory to php-todo directory. Deploy the containers:
```
docker-compose up -d
```
![{6E56E0A6-D362-4ED3-ABB6-C6B097B2BF50} png](https://user-images.githubusercontent.com/76074379/137554026-2a2d501d-98ac-4d81-88d9-d629acb1c83e.jpg)

- We are going to open a docker hub account if we do not have already. Go to  your bbroswer and open a dockerhub account
- On your terminal/editor, create a new tag for the image you want to push using the
proper syntax.

```
docker tag php-todo:latest thecountt/php-todo:1.0.0
```
![{A380BC13-5CC6-40ED-A34C-00FCC569E189} png](https://user-images.githubusercontent.com/76074379/137539643-637ca9c4-7737-433b-97c5-688db1dfed1d.jpg)
- Run this command to see the image with the newly created tag

```
docker ps -a
```

Login to your dockerhub account and type in your credentials

```
docker login
```
![{C8CA5105-2713-4511-9AEC-A46B41BFB7E1} png](https://user-images.githubusercontent.com/76074379/137540418-2a9d026d-89fb-4276-968d-95961566f74d.jpg)

- Push the docker image from the local machine to the dockerhub repository
```
docker push thecountt/php-todo:1.0.0
```
![{80B6ABB2-FF9E-48E7-84CD-6FECF1CD5143} png](https://user-images.githubusercontent.com/76074379/137539999-fa21ec99-e1ac-4c4c-a609-bb666eaafa2f.jpg)

## CI/CD with Jenkins (Machine or Container)

### 1. Using Local Machine

-Stop and remove the manually deployed containers of above
```
docker-compose down
```

- Run the following command in your home directory to install java runtime:
```
sudo apt update -y
sudo apt install openjdk-11-jdk
```
- Run the following commands to install jenkins:
```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

#### Unlocking Jenkins
- When you first access a new Jenkins instance, you are asked to unlock it using an automatically-generated password.

- Browse to http://localhost:8080 (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears and you can use

`sudo cat /var/lib/jenkins/secrets/initialAdminPassword` to print the password on the terminal.

![{19CD1653-CEAF-4F90-81E5-256330E7448E} png](https://user-images.githubusercontent.com/76074379/137549596-27a14fa9-9a86-4039-bfc4-8be9adb1a76a.jpg)

#### Jenkins Pipeline
- First we will install the plugins needed
  - On the Jenkins Dashboard, click on `Manage Jenkins` and go to `Manage Plugins`.
  - Search and install the following plugins:
    - Blue Ocean
    - Docker
    - Docker Compose Build Steps
    - HttpRequest
    
- Create a new repository in your dockerhub account to push image into

![{6782567F-DE5D-4C06-83D8-07E392117EFE} png](https://user-images.githubusercontent.com/76074379/137549931-c60f0b5b-4fb8-4fa1-af38-33404263e198.jpg)

- We need to create credentials that we will reference so as to be able to push our image to the docker hub repository

  - Click on  `Manage Jenkins` and go to `Manage Credentials`.
  - Click on `global`
  - Click on `add credentials` and choose `username with password`
  - Input your dockerhub username and password

![{F069636D-EE64-4502-B73A-781C822FC9F8} png](https://user-images.githubusercontent.com/76074379/137545904-b1fa454c-cb1d-4866-bd97-157fd7a4f143.jpg)

- Create a Jenkinsfile in the php-todo directory that will build image from context in the github repo; deploy application; make http request to see if it returns the status code 200 & push the image to the dockerhub repository and finally clean-up stage where the  image is deleted on the Jenkins server

```
pipeline {
    environment {
    REGISTRY = credentials('docker-hub-cred-2')
    PATH = "$PATH:/usr/bin"

    }
    
    agent any

    stages {

        stage('Initial Cleanup') {
            steps {
                dir("${WORKSPACE}") {
                deleteDir()
                }
            }
        }
  
        stage('Checkout SCM') {
            steps {
                git branch: 'main', url: 'https://github.com/TheCountt/docker-php-todo.git'
            }
        }
        
        stage('Build image') {
            steps {
                sh "docker build -t thecountt/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER} ."
            }
        }
        stage("Start application") {
            steps {
                sh "docker-compose --version"
                sh "docker-compose up -d"
            }
        }
        stage("Test Endpoint and Push Image to Registry") {
            steps {
                script {
                    while (true) {
                        def response = httpRequest 'http://localhost:8000'
                        if (response.status == 200) {
                                sh "docker push thecountt/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                            break 
                        }
                    }
                }
            }
        }
 
        stage ("Remove images") {
            steps {
	    	sh "docker-compose down"
                sh "docker rmi thecountt/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
            }
        }
    }
}
```
![{71A29707-09CD-4F04-B2BE-5CE000F75B4C} png](https://user-images.githubusercontent.com/76074379/137603739-a6a2fa21-07e8-4fcb-9bc4-1a946f3b998b.jpg)


**Note: the docker compose package is in `usr/bin`, that is why it is specified in the jenkinsfile**

- Click on "Scan Repository Now" 

- A build will start. The pipeline should be successful now


![{1E38ABFB-5A45-4D94-A9C6-B95015D8E35F} png](https://user-images.githubusercontent.com/76074379/137547662-f7d58d0c-c95a-465c-ac3d-952b235ec1d4.jpg)

![{C9FB3404-990A-4F72-9F98-893661BE041F} png](https://user-images.githubusercontent.com/76074379/137547562-13fb1d7f-05ab-4dde-8cf4-13aed5e22ab0.jpg)

#### Github Webhook
We need to create  a webhook so that Jenkins will automatically pick up changes in our github repo and trigger a build instead of havinf to click "Scan Repository Now" all the time on jenkins. However, we cannot connect to our localhost because it i in a private network. We will have to use a proxy server. We will map our localhost to our proxy server. The proxy server will then generate a URL for us. We will input that URL in github webhooks so any changes we make to our github repo will automatically trigger a build.

We will use **localtunnel**, a nodejs proxy server

- Install nodejs npm nodejs-legacy if you do not have it installed already on your local machine
```
sudo apt install nodejs npm nodejs-legacy
```

- Install Localtunnel

```
npm install -g localtunnel
```
![{47A4268B-D9A9-453A-A7AE-26A6D947006B} png](https://user-images.githubusercontent.com/76074379/137554552-04197c7f-de48-4c4a-af8d-826b95715909.jpg)

- Run the following command to get our unique url mapped to our jenkins port

```
lt --port 8080 --subdomain docker-projects
```
![{984D3080-5EDD-463C-8421-CE5D7DAF728A} png](https://user-images.githubusercontent.com/76074379/137554885-5712c622-89d5-4d98-a1d1-5c596316d350.jpg)

- Go to github repository and click on `Settings`
	- Click on `Webhooks`
	- Click on `Add Webhooks`
	- Input the generated URL with /postreceive as shown in the Payroad URL space
	- Select application/json as the Content-Type
	- Click on `Add Webhook` to save the webhook

![{EF2FCB11-2154-44E8-BF4F-7F724E5DA095} png](https://user-images.githubusercontent.com/76074379/137589297-5b6e45d5-1dc6-40a7-a10b-294c951ce82f.jpg)

- Go to your terminal and change something in your jenkinsfile and save and push to your github repo. If everything works out fine, this will trigger a build which you can see on your Jenkins Dashboard.

![{C5521131-DCAF-4DFB-A3FC-36D6770938D6} png](https://user-images.githubusercontent.com/76074379/137589378-1c02fab0-e171-4b14-b8e7-65b9416a87f4.jpg)

![{1739C86D-67AC-4BC6-B48E-7D2CBA9685A8} png](https://user-images.githubusercontent.com/76074379/137589448-187d860b-852e-4862-a0e5-05be21b5f5b3.jpg)


**Note: Your localtunnel generated URL might be unable to load on your browser if you do not specify the HTTPS port in the URL. So you may do this `generated URL:443 e.g https://docker-experiment.loca.lt:443. Though, it is strongly adviced never to use this strategy for anything that has personally identifying information or anything sensitive.
The best way to use localtunnel is to build your own server because that is far safer. To do that, click [here](https://github.com/localtunnel/server#deploy)**

### 2. Using Docker Container
- Create a directory and name it `jenkins` and change into the directory.
- Create a bridge network in Docker using the following docker network create command. We will use the network we have created earlier(tooling_app_network)

- In order to execute Docker commands inside Jenkins nodes, download  the `docker:dind` Docker image so we can use it in our docker compose file we will create:

```
docker pull docker:dind
```

- Customise official Jenkins Docker image, by executing the two steps below:

    a. Create Dockerfile with the following content:

```
FROM jenkins/jenkins:2.303.1-jdk11
USER root
RUN apt-get update && apt-get install -y apt-transport-https \
       ca-certificates curl gnupg2 \
       software-properties-common
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
RUN apt-key fingerprint 0EBFCD88
RUN add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/debian \
       $(lsb_release -cs) stable"
RUN curl -L \  
  "https://github.com/docker/compose/releases/download/v2.0.0-beta.6/docker-compose-$(uname -s)-$(uname -m)" \  
  -o /usr/bin/docker-compose \  
  && chmod +x /usr/bin/docker-compose
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.0 docker-workflow:1.26"

```
![{4E1248AE-F360-4413-9F2E-CA66975FBD37} png](https://user-images.githubusercontent.com/76074379/137628891-414df2b8-d62a-412e-99c5-8d2d56506df9.jpg)

   b. Build a new docker image from this Dockerfile and assign the image a meaningful name, e.g. "myjenkins-blueocean:1.1":
```
docker build -t myjenkins-blueocean:1.1 . 
```
![{DDD93499-A8DB-4D40-858D-C3FD470F4D10} png](https://user-images.githubusercontent.com/76074379/137629226-8082b26b-f1f4-4f21-b21e-61219476eb41.jpg)

- Create a file and name it jenkins.yml. Paste the code below:

```
version: "3.9"
services:
    docker:
        image: "docker:dind"
        container_name: jenkins-docker
        privileged: true
        network_mode: tooling_app_network
        
        environment:
            - DOCKER_TLS_CERTDIR=/certs
            - DOCKER_DRIVER=overlay2
        volumes:
            - 'jenkins-docker-certs:/certs/client'
            - 'jenkins-data:/var/jenkins_home'
        ports:
            - "2376:2376"

    myjenkins-blueocean:
        image: "myjenkins-blueocean:1.1"
        container_name: jenkins-blueocean
        network_mode: tooling_app_network
        environment:
            - DOCKER_HOST=tcp://docker:2376
            - DOCKER_CERT_PATH=/certs/client
            - DOCKER_TLS_VERIFY=1
        ports:
            - "8080:8080"
            - "50000:50000"
        
        volumes:
            - "jenkins-data:/var/jenkins_home"
            - "jenkins-docker-certs:/certs/client:ro"
        
volumes:
   docker:
   myjenkins-blueocean:
   jenkins-docker-certs:
   jenkins-data:
```
![{429A6E3A-590F-4087-A4AA-CC9656321D87} png](https://user-images.githubusercontent.com/76074379/137629108-78974dd5-9dca-41ff-be44-d44a0a46889e.jpg)

- Spin up jenkins container:
```
docker-compose -f jenkins.yml up -d
```
![{0B77D6FC-5517-446F-8F9F-96456F3CD51F} png](https://user-images.githubusercontent.com/76074379/137629395-c5777a2c-6d28-4358-86a0-2b7ac18eff01.jpg)

### Unlocking Jenkins
- When you first access a new Jenkins instance, you are asked to unlock it using an automatically-generated password.

- Browse to http://localhost:8080 (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears

- If you are running Jenkins in Docker using the official jenkins/jenkins image you can use:

`sudo docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword` to print the password in the console without having to exec into the container.


### Jenkins Pipeline

- On Jenkins UI. Go to `Manage Plugins` and install the following plugins: **HttpRequest; Docker; Docker Compose Build Steps**

- Create a new repository in your dockerhub account to push image into

- Create a Jenkinsfile in the php-todo directory. Write a Jenkinsfile that will pick up the build context from the referenced github repo branch, deploy application; make http request to see if it returns the status code 200 & push the image to the dockerhub repository and finally clean-up stage where the image is deleted on the Jenkins server


```
pipeline {
    environment {
        registry = "thecountt/docker-php-todo"
        registryCredential = "docker-hub-cred"
	PATH = "$PATH:/usr/local/bin"
    }
    
    agent any
    stages {
        
        stage('Initial Cleanup') {
          steps {
                dir("${WORKSPACE}") {
                    deleteDir()
                }
            }
        }

        
        stage('Cloning Git repository') {
          steps {
                git branch : 'main', url: 'https://github.com/TheCountt/docker-php-todo.git'
            }
        }

        
        stage('Build Image') {
            steps{
                script {
                    dockerImage = docker.build registry + ":$BUILD_NUMBER"
                }
            }
        }

        
        stage("Start the app") {
            steps {
	      sh 'docker-compose --version'
              sh 'docker-compose -f jenkins.yml up -d'
            }
    }	
      
      stage("Test endpoint") {
            steps {
                script {
                    while (true) {
                        def response = httpRequest 'http://localhost:8000'

                        if (http.responseCode == 200) {
                        sh 'echo "httpRequest Successsful"'
                        break
                        }
                    }
                }
            }
        }


        stage('Push Image') {
            steps{
                script {
                    docker.withRegistry( '', registryCredential ) {
                        dockerImage.push()
                    }
                }
            }
        }

        
        stage('Remove Unused docker image') {
            steps{
                sh "docker rmi $registry:$BUILD_NUMBER"
            }
        }
    }
}
```
![{074CD953-FFF9-4541-AA56-BA4C930AF3C1} png](https://user-images.githubusercontent.com/76074379/137629996-aa882bfe-37dc-4d68-819a-be290f2a89fa.jpg)


- Create a  Pipeline using Blue Ocean.

- If you run a build, the pipeline will fail because we have not configured our dockerhub
(registryCredential) in jenkins.
  a. Go to jenkins home, click on your pipeline. On the left-hand side, click on "configure"
  b. Click on "Properties".
  c. Fill the "Docker registry URL"
  c. In the "Registry credentials", click on "Add". Put your dockerhub account credentials
     (Username and Password) and save it.

![{F069636D-EE64-4502-B73A-781C822FC9F8} png](https://user-images.githubusercontent.com/76074379/137630083-3b93691d-c8e6-43f5-86a0-04a5d286976c.jpg)

- We need to change the name of the path of the Jenkinsfile correctly on Jenkins or Jenkins pipeline will not run
	a. Click on `Dashboard`
	b. To the left, click on `Configure` and Scroll down to `Build Confugration` and make sure the path to the Jenkinsfile and is correct.
	c. Click 'Save` at the bottom.

- Click on "Scan Repository"

- Run a build now. The  pipeline should be successful

- We have to create a github webhook so jenkins can automatically pickup changes and run a build. But we cannot connect to a localhost in a private network. So we will use a proxy server called **localtunnel.**

- Create a folder and name it localtunnel. Create a file in the folder and name it Dockerfile.lt
- Paste the following code in the Dockerfile.lt
```
FROM node:lts-alpine3.14

RUN npm install -g localtunnel

ENTRYPOINT ["lt"]
```

- Go to the jenkins directory and in our jenkins.yml file, paste the localtunnel block of code into the file

```
version: "3.9"
services:
    docker:
        image: "docker:dind"
        container_name: jenkins-docker
        privileged: true
        network_mode: tooling_app_network
        
        environment:
            - DOCKER_TLS_CERTDIR=/certs
            - DOCKER_DRIVER=overlay2
        volumes:
            - 'jenkins-docker-certs:/certs/client'
            - 'jenkins-data:/var/jenkins_home'
        ports:
            - "2376:2376"

    myjenkins-blueocean:
        image: "myjenkins-blueocean:1.1"
        container_name: jenkins-blueocean
        network_mode: tooling_app_network
        environment:
            - DOCKER_HOST=tcp://docker:2376
            - DOCKER_CERT_PATH=/certs/client
            - DOCKER_TLS_VERIFY=1
        ports:
            - "8080:8080"
            - "50000:50000"
        
        volumes:
            - "jenkins-data:/var/jenkins_home"
            - "jenkins-docker-certs:/certs/client:ro"
        

    localtunnel:
        build:
          context: ~/localtunnel
          dockerfile: Dockerfile.lt
        network_mode: tooling_app_network
        
        command: --local-host myjenkins-blueocean --port 8080 --subdomain jenkins-container

volumes:
   docker:
   myjenkins-blueocean:
   jenkins-docker-certs:
   jenkins-data:
```


- Go to the Jenkins dashboard and click on "Scan Repository Now" to trigger a build
- Go to your terminal and run command 	`docker ps -a`. Copy the name of the or ID of the localtunnel.
- Pass the name or the ID of the localtunnel container in this command below to get the generated URL
```
docker logs <name-of-container or ID>
```
- Copy the generated URL and load it on a browser to see if it is reachable
- Go to github repository and click on `Settings`
	- Click on `Webhooks`
	- Click on `Add Webhooks`
	- Input the generated URL with /postreceive as shown in the Payroad URL space
	- Select application/json as the Content-Type
	- Click on `Add Webhook` to save the webhook



- Go to your terminal and change something in your jenkinsfile and save and push to your github repo. If everything works out fine, this will trigger a build which you can see on your Jenkins Dashboard.








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

## CI/CD with Jenkins - Deploying/Building Docker Containers & Pushing to Dockerhub using Jenkins

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

- Browse to http://ip-address:8080 (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears and you can use

`sudo cat /var/lib/jenkins/secrets/initialAdminPassword` to print the password on the terminal.

#### Jenkins Pipeline
- First we will install the plugins needed
  - On the Jenkins Dashboard, click on `Manage Jenkins` and go to `Manage Plugins`.
  - Search and install the following plugins:
    - Blue Ocean
    - Docker
    - Docker Compose Build Steps
    - HttpRequest

- We need to create credentials that we will reference so as to be able to push our image to the docker hub repository

  - Click on  `Manage Jenkins` and go to `Manage Credentials`.
  - Click on `global`
  - Click on `add credentials` and choose `username with password`
  - Input your dockerhub username and password

- Create a Jenkinsfile in the php-todo directory that will build image from context in the github repo; deploy application; make http request to see if it returns the status code 200 & push the image to the dockerhub repository and finally clean-up stage where the  image is deleted on the Jenkins server

```
pipeline {
    environment {
        REGISTRY = credentials('dockerhub-cred')
    }
    agent any

    stages{

        stage('Initial Cleanup') {
            steps {
                dir("${WORKSPACE}") {
                deleteDir()
                }
            }
        }

        stage('Checkout SCM') {
            steps {
                git branch: 'master', url: 'https://github.com/brpo01/docker-todo-webapp.git'
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t tobyrotimi/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER} ."
            }
        }

        stage('Start the application') {
            steps {
                sh "docker-compose up -d"
            }
        }

        stage('Test endpoint & Push Image to Registry') {
            steps{
                script {
                    while(true) {
                        def response = httpRequest 'http://localhost'
                        if (response.status == 200) {
                            withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                                sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPassword}"
                                sh "docker push tobyrotimi/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                            }
                            break 
                        }
                    }
                }
            }
        }

        stage('Remove Images') {
            steps {
                sh "docker-compose down"
                sh "docker rmi tobyrotimi/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
            }
        }
    }
}
```

- Go To Jenkins Blue Ocean & trigger a build.

- A build will start. The pipeline should be successful now

![20](https://user-images.githubusercontent.com/47898882/145481897-1dae5b91-9140-41c3-9aac-53a7d8abe839.JPG)

#### Github Webhook
We need to create  a webhook so that Jenkins will automatically pick up changes in our github repo and trigger a build instead of having to click "Scan Repository Now" all the time on jenkins. However, we cannot connect to our localhost because it i in a private network. We will have to use a proxy server. We will map our localhost to our proxy server. The proxy server will then generate a URL for us. We will input that URL in github webhooks so any changes we make to our github repo will automatically trigger a build.


- Go to github repository and click on `Settings`
	- Click on `Webhooks`
	- Click on `Add Webhooks`
	- Input the generated URL with /postreceive as shown in the Payroad URL space
	- Select application/json as the Content-Type
	- Click on `Add Webhook` to save the webhook

- Go to your terminal and change something in your jenkinsfile and save and push to your github repo. If everything works out fine, this will trigger a build which you can see on your Jenkins Dashboard.

![20](https://user-images.githubusercontent.com/47898882/145481897-1dae5b91-9140-41c3-9aac-53a7d8abe839.JPG)



## 2. PHP-TODO Application Containerization using Docker

Repo: https://github.com/brpo01/docker-todo-webapp

- Clone php-todo repository using `wget`(and unzip it) or `git clone`

- Write a dockerfile for php-todo app and save it in the php-todo directory
```
FROM php:7-apache
MAINTAINER Rotimi opraise139@gmail.com

# Install docker php dependencies
RUN docker-php-ext-install mysqli 

# copy virtualhost config file unto the apache deafult conf in the container
COPY apache-config.conf /etc/apache2/sites-available/000-default.conf
COPY start-apache /usr/local/bin
RUN a2enmod rewrite

# Install php dependency - composer
RUN curl -sS https://getcomposer.org/installer |php && mv composer.phar /usr/local/bin/composer

#Copy all files 
COPY . /var/www/html
RUN chown -R www-data:www-data /var/www

EXPOSE 80

CMD ["start-apache"]
```

- Open the `start-apache` file, add the following commands below in addition to the commands already in the file:

```
#!/usr/bin/env bash
sed -i "s/Listen 80/Listen ${PORT:-80}/g" /etc/apache2/ports.conf
sed -i "s/:80/:${PORT:-80}/g" /etc/apache2/sites-enabled/*

composer install --no-plugins --no-scripts

php artisan migrate
php artisan key:generate
php artisan db:seed

apache2-foreground
```

- Create a docker-compose.yml in the php-todo directory and paste the code below:

```yaml
version: "3.3"

services: 
  todo-app:
    build: .
    links:
      - todo-db
    depends_on: 
      - todo-db
    restart: always
    volumes:
      - todo-app:/var/www/html
    ports:
      - ${APP_PORT}:80
  todo-db:
    image: mysql:5.7
    hostname: ${MYSQL_HOSTNAME}
    volumes:
      - todo-db:/var/lib/mysql
    restart: always
    environment:
      MYSQL_DATABASE: ${MYSQL_DB}
      MYSQL_USERNAME: ${MYSQL_USERNAME}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}

volumes:
  todo-app:
  todo-db:
```

- Update the `.env` file

```
APP_ENV=local
APP_DEBUG=true
APP_KEY=SomeRandomString
APP_URL=http://localhost

MYSQL_HOSTNAME=mysqlserverhost
MYSQL_DB=homestead
MYSQL_USERNAME=homestead
MYSQL_PASSWORD=sePret^i
MYSQL_ROOT_PASSWORD=password1234567
APP_PORT=8000

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

- Make sure you change directory to php-todo directory. Build image using this command:
```
docker build -t php-todo:latest .
```

- Make sure you change directory to php-todo directory. Deploy the containers:
```
docker-compose up -d
```
![16](https://user-images.githubusercontent.com/47898882/145481880-aff9ae4d-cd12-4063-ac02-1501276823fe.JPG)

- We are going to open a docker hub account if we do not have already. Go to  your bbroswer and open a dockerhub account
- On your terminal/editor, create a new tag for the image you want to push using the
proper syntax.

```
docker tag php-todo:latest tobyrotimi/php-todo:1.0.0
```

- Run this command to see the image with the newly created tag

```
docker ps -a
```

Login to your dockerhub account and type in your credentials

```
docker login
```

![18](https://user-images.githubusercontent.com/47898882/145481888-79739991-eb03-461d-9b0a-9d295569c41b.JPG)

- Push the docker image from the local machine to the dockerhub repository
```
docker push tobyrotimi/php-todo:1.0.0
```

![17](https://user-images.githubusercontent.com/47898882/145481885-fcd096d6-4e39-4d95-8910-af9e2d952a5b.JPG)

![19](https://user-images.githubusercontent.com/47898882/145481892-9da33784-9efe-451b-81c1-3864ebb420d2.JPG)

## CI/CD with Jenkins (Machine or Container) - Deploying/Building Docker Containers & Pushing to Dockerhub using Jenkins

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

- Browse to http://ip-address:8080 (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears and you can use

`sudo cat /var/lib/jenkins/secrets/initialAdminPassword` to print the password on the terminal.

#### Jenkins Pipeline
- First we will install the plugins needed
  - On the Jenkins Dashboard, click on `Manage Jenkins` and go to `Manage Plugins`.
  - Search and install the following plugins:
    - Blue Ocean
    - Docker
    - Docker Compose Build Steps
    - HttpRequest

- We need to create credentials that we will reference so as to be able to push our image to the docker hub repository

  - Click on  `Manage Jenkins` and go to `Manage Credentials`.
  - Click on `global`
  - Click on `add credentials` and choose `username with password`
  - Input your dockerhub username and password

- Create a Jenkinsfile in the php-todo directory that will build image from context in the github repo; deploy application; make http request to see if it returns the status code 200 & push the image to the dockerhub repository and finally clean-up stage where the  image is deleted on the Jenkins server

```
pipeline {
    environment {
        REGISTRY = credentials('dockerhub-cred')
    }
    agent any

    stages{

        stage('Initial Cleanup') {
            steps {
                dir("${WORKSPACE}") {
                deleteDir()
                }
            }
        }

        stage('Checkout SCM') {
            steps {
                git branch: 'master', url: 'https://github.com/brpo01/docker-todo-webapp.git'
            }
        }

        stage('Build Image') {
            steps {
                sh "docker build -t tobyrotimi/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER} ."
            }
        }

        stage('Start the application') {
            steps {
                sh "docker-compose up -d"
            }
        }

        stage('Test endpoint & Push Image to Registry') {
            steps{
                script {
                    while(true) {
                        def response = httpRequest 'http://localhost'
                        if (response.status == 200) {
                            withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
                                sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPassword}"
                                sh "docker push tobyrotimi/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
                            }
                            break 
                        }
                    }
                }
            }
        }

        stage('Remove Images') {
            steps {
                sh "docker-compose down"
                sh "docker rmi tobyrotimi/docker-php-todo:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
            }
        }
    }
}
```

- Go To Jenkins Blue Ocean & trigger a build.

- A build will start. The pipeline should be successful now

![20](https://user-images.githubusercontent.com/47898882/145481897-1dae5b91-9140-41c3-9aac-53a7d8abe839.JPG)

#### Github Webhook
We need to create  a webhook so that Jenkins will automatically pick up changes in our github repo and trigger a build instead of having to click "Scan Repository Now" all the time on jenkins. However, we cannot connect to our localhost because it i in a private network. We will have to use a proxy server. We will map our localhost to our proxy server. The proxy server will then generate a URL for us. We will input that URL in github webhooks so any changes we make to our github repo will automatically trigger a build.


- Go to github repository and click on `Settings`
	- Click on `Webhooks`
	- Click on `Add Webhooks`
	- Input the generated URL with /postreceive as shown in the Payroad URL space
	- Select application/json as the Content-Type
	- Click on `Add Webhook` to save the webhook

- Go to your terminal and change something in your jenkinsfile and save and push to your github repo. If everything works out fine, this will trigger a build which you can see on your Jenkins Dashboard.

![20](https://user-images.githubusercontent.com/47898882/145481897-1dae5b91-9140-41c3-9aac-53a7d8abe839.JPG)






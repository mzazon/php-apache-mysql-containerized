Containerize This: PHP/Apache/MySQL
===================================

### Intro
Continuing with the Containerize This! series, we're looking at common web application technologies and how they can be used within Docker containers effectively. PHP/Apache/MySQL have a very large market share on content management systems and web applications on the internet, and with so many developers using these technologies, there is a lot of interest to modernize the way that they use them from from local development all the way to production. Today we'll take a look at several ways to containerize and link PHP, Apache, and MySQL together while demonstrating some tips, tricks, and best-practices that will help you take a modern approach when developing and deploying your PHP applications!

There are 5 simple files for this demo that you can clone from https://github.com/mzazon/php-apache-mysql or simply copy and paste from this post to replicate the following folder structure. Please note that some Docker and security principals have been skipped here for simplicity and demonstration purposes. Namely PHP using root credentials, hardcoded/weak MySQL password, lack of SSL, to name a few! Do not run this code in production! :-)

```
/php-apache-mysql/
├── apache
│   ├── Dockerfile
│   └── demo.apache.conf
├── docker-compose.yml
├── php
│   └── Dockerfile
└── public_html
    └── index.php
```

Once this structure is replicated or cloned with these files, and Docker installed locally, you can simply run "docker-compose up" from the root of the project to run this entire demo, and point your browser (or curl) to http://localhost:80 to see the demo. We will get into what "docker-compose" is, and what makes up this basic demonstration in the following sections!

We'll use the following simple PHP application to demonstrate everything:

#### index.php
```
<h1>Hello Cloudreach!</h1>
<h4>Attempting MySQL connection from php...</h4>
<?php
$host = 'mysql';
$user = 'root';
$pass = 'rootpassword';
$conn = new mysqli($host, $user, $pass);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
} else {
  echo "Connected to MySQL successfully!";
}
?>
```
This code attempts to connect to a MySQL database using the mysqli interface from PHP. If successful, it prints a success. If not, it prints a failed message. 

### Docker Compose

This format has been around for a while in Dockerland and is now in version 3.6 at the time of this writing. We'll use 3.2 here to ensure broad compatibility with those who may not be running the latest and greatest versions of Docker (however, you should always upgrade!)

This format allows for defining sets of services which make up an entire application. It allows you to define the dependencies for those services, networks, volumes, etc as code and as you roll into production, you can even specify deployment parameters on services which allow you to replicate, scale, update, and self-heal on Docker Swarm or even Kubernetes on the latest Docker for Mac or Docker Enterprise Edition!

#### docker-compose.yml
```
version: "3.2"
services:
  php:
    build: 
      context: './php/'
      args:
       PHP_VERSION: ${PHP_VERSION}
    networks:
      - backend
    volumes:
      - ${PROJECT_ROOT}/:/var/www/html/
    container_name: php
  apache:
    build:
      context: './apache/'
      args:
       APACHE_VERSION: ${APACHE_VERSION}
    depends_on:
      - php
      - mysql
    networks:
      - frontend
      - backend
    ports:
      - "80:80"
    volumes:
      - ${PROJECT_ROOT}/:/var/www/html/
    container_name: apache
  mysql:
    image: mysql:${MYSQL_VERSION:-latest}
    restart: always
    ports:
      - "3306:3306"
    volumes:
            - data:/var/lib/mysql
    networks:
      - backend
    environment:
      MYSQL_ROOT_PASSWORD: "${DB_ROOT_PASSWORD}"
      MYSQL_DATABASE: "${DB_NAME}"
      MYSQL_USER: "${DB_USERNAME}"
      MYSQL_PASSWORD: "${DB_PASSWORD}"
    container_name: mysql
networks:
  frontend:
  backend:
volumes:
    data:
```

### Decouple dependencies, run one process per container (PID 1)
Containers at the core are simply process encapsulation within a shared Linux kernel, to allow for many processes to run inside of their own spaces without interfering or interacting unless we explicitly tell them they can. Thus, it is a long-running best practice to run your primary service within a container as PID 1. This means that no other process should be running inside of your containers. It allows containers to maintain a native linux process lifecycle, respect linux process signals, allow for greater security, and will make your life easier when you schedule these containers on orchestrators such as Docker Swarm or Kubernetes in production.

A perfect example is decoupling Apache and PHP by building them out into separate containers. We see our customers often starting to couple Apache and PHP together early on in a Docker journey by building custom images which include both Apache and PHP in the image. This easily works in development scenarios and is a fast way to get off the ground, but as we want to follow a more modern approach of decoupling, we want to break these apart.

The following simple Dockerfiles are what we're using in this example to build a decoupled Apache and PHP envivonment:

#### apache/Dockerfile
```
FROM httpd:2.4.33-alpine

RUN apk update; \
    apk upgrade;

# Copy apache vhost file to proxy php requests to php-fpm container
COPY demo.apache.conf /usr/local/apache2/conf/demo.apache.conf
RUN echo "Include /usr/local/apache2/conf/demo.apache.conf" \
    >> /usr/local/apache2/conf/httpd.conf
```

#### php/Dockerfile
```
FROM php:7.2.7-fpm-alpine3.7

RUN apk update; \
    apk upgrade;

RUN docker-php-ext-install mysqli
```

Note that we run minimal containers wherever possible, in this example we're using official alpine-based images!

### Networking
Now that we have container images for Apache and PHP that are decoupled, how do we get these to interact with eachother? Notice we're using "PHP FPM" for this. We're going to have Apache proxy connections which require PHP rendering to port 9000 of our PHP container, and then have the PHP container serve those out as rendered HTML. Sound complicated? Don't worry! This is very common in modern applications, Apache and NGINX are very good at this proxying, and there is plenty of documentation out there to support this behavior!

Notice in the above Docker Compose example, we link the containers together with an overlay networks we define as "frontend" and "backend". By specifying these networks in our services, we can leverage some really cool Docker features! Namely, we can refer to the containers/services by their service names in code, so we don't have to worry about messy hard-coding of IP addresses anymore. Phew! Docker handles this for us by managing and updating /etc/hosts files within containers seamlessly to allow for this cross-talk. We can also enforce which containers can talk to eachother. This allows for more secure applications. Notice in this example we have "frontend" and "backend" defined. We can have our php and mysql services live in a network that is more restricted and not exposed to the outside world, and only expose our apache container on the required port. This is great for production environments!

One note: We need an apache vhost configuration file that is set up to proxy these requests for PHP files to the PHP container. You'll notice in the Dockerfile for Apache we have defined above, we add this file and then include it in the base httpd.conf file. This is for the proxying which allows for the decoupling of Apache and PHP. In this example we called it demo.apache.conf and we have the proxying modules defined as well as the VirtualHost. Also notice that we call the php container in the code "php" which, because of the /etc/hosts file integration that Docker handles for us, works flawlessly!

#### apache/demo.apache.conf
```
ServerName localhost

LoadModule deflate_module /usr/local/apache2/modules/mod_deflate.so
LoadModule proxy_module /usr/local/apache2/modules/mod_proxy.so
LoadModule proxy_fcgi_module /usr/local/apache2/modules/mod_proxy_fcgi.so

<VirtualHost *:80>
    # Proxy .php requests to port 9000 of the php-fpm container
    ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php:9000/var/www/html/$1
    DocumentRoot /var/www/html/
    <Directory /var/www/html/>
        DirectoryIndex index.php
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # Send apache logs to stdout and stderr
    CustomLog /proc/self/fd/1 common
    ErrorLog /proc/self/fd/2
</VirtualHost>
```

### Volumes
The last feature that we'll call out here for demonstration purposes is the use of volumes to serve out our code. Both the PHP and Apache containers have access to a "volume" that we define in the docker-compose.yml file which maps the public_html folder of our repository to the respective services for them to access. When we do this, we map a folder on the host filesystem (outside of the container context) to inside of the running containers. Developers may wish to set their projects up like this because it allows them to edit the file outside of the container, yet have the container serve the updated PHP code out as soon as changes are saved.

Volumes are a very powerful construct of the Docker world and we're only scratching the surface of what one can achieve by using them in development and production. Please see the official documentation on volumes for further use cases and best practices!

### Demonstration of docker-compose up!

```
$ docker-compose up

<... some details omitted ...>
php_1     | [16-Jul-2018 02:08:11] NOTICE: fpm is running, pid 1
php_1     | [16-Jul-2018 02:08:11] NOTICE: ready to handle connections
apache_1  | [Mon Jul 16 02:08:12.494294 2018] [pid 1:tid 140290664872840] AH00489: Apache/2.4.33 (Unix)
apache_1  | [Mon Jul 16 02:08:12.496833 2018] [pid 1:tid 140290664872840] AH00094: Command line: 'httpd -D FOREGROUND'
mysql_1   | 2018-07-16 02:08:12 1 [Note] mysqld: ready for connections.
mysql_1   | Version: '5.6.40'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server (GPL)
```
Notice how these 3 daemons run on PID 1 inside of each container, this is considered a best-practice for building containers!

```
$  curl localhost:80
Hello Cloudreach!
Attempting MySQL connection from php...
Connected to MySQL successfully!
```

Notice now in the logs that the apache container and php containers both respond to the request, as desired:
```
php_1     | 172.18.0.4 -  16/Jul/2018:02:09:22 +0000 "GET /index.php" 200
apache_1  | 172.18.0.1 - - [16/Jul/2018:02:09:22 +0000] "GET / HTTP/1.1" 200 108
```

### Conclusion

With these basic principals you can link services together to create applications. You could easily include "composer" in the PHP container to build and run your PHP/Laravel application in a similar manner. Perhaps you want to run Drupal or Wordpress and decouple PHP from the Apache instance, that is possible too! You can even use this to seamlessly test PHP version or MySQL version upgrades with minimal code change. There are a lot of benefits to modernizing your application with Docker using docker-compose and some of the latest images and features.

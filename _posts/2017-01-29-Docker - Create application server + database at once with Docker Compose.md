---
layout: post
title:  "Docker - Create application server + database at once with Docker Compose"
date:   2017-01-29 21:30:01 +0200
categories: docker glassfish postgres
---
In a previous post we were discussing about glassfish installation in a VM at ~okeanos or Digital Ocean.

But when we start developing we might need also a database side by side our application server. Docker Compose is very useful, especially if we don't want to install all these components to our system or to maximise portability. Compose is a tool that is used to define and run multiple Docker applications.

<h4>Install Docker Compose</h4>

Before using Docker Compose make sure you have installed it on your system. Instructions based on your system are provided at <a href="https://docs.docker.com/compose/install/">Docker Compose</a>.

<strong>Create docker-compose file and </strong><b>organise directories</b>

To begin with, first create a file named <strong>docker-compose.yml</strong> and place it where you need your services to take place.
<pre><strong>$docker&gt;</strong>mkdir services
<strong>$docker&gt;</strong>cd services
<strong>$services&gt;</strong>touch docker-compose.yml</pre>
To organise our services, we will create two separate directories, one for the database and one for the application server.
<pre><strong>$services&gt;</strong>mkdir db
<strong>$services&gt;</strong>mkdir ws</pre>
<strong>Database setup</strong>
<br>
Lets begin with the database. For the purposes of this post we use postgres but likewise other databases such as mySQL, mongoDB etc are available in the Docker repositories.
<br>
Also, we need to create an sql script that contains the schema of the database. Lets create one and place it within the db directory:
<pre><strong>$services&gt;</strong>cd db
<strong>$db&gt;</strong>touch sample-setup.sql</pre>
With a text editor or pico/nano open sample.sql and paste the following lines:
<pre>-- sample db schema
CREATE DATABASE sample;
CREATE TABLE IF NOT EXISTS user (
  key   SERIAL PRIMARY KEY,
  username  CHAR(50) NOT NULL,
  password CHAR(50) NOT NULL
);

CREATE TABLE IF NOT EXISTS customer (
  key           SERIAL PRIMARY KEY,
  user_key   INTEGER references user(key),
  first_name    CHAR(50) NOT NULL,
  last_name     CHAR(50) NOT NULL
);</pre>
Save the file and exit. Then with a text editor or pico/nano open the docker-compose file and set the following lines:
<pre>version: '2'
services:
  db:
    image: library/postgres
    volumes:
      - ./db/pgdata:/pgdata
      - ./db/sample-setup.sql:/docker-entrypoint-initdb.d/sample-setup.sql
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=docker
      - POSTGRES_PASSWORD=docker
      - PGDATA=/pgdata
</pre>
<strong>version: '2'</strong> signifies Compose version
<strong>services:</strong> this is where the section of the services starts. The lines in the yml file need to have the proper indentation for each section. So, all lines after a section, the line with the colon (:), should be indented by two spaces.
<br>
<strong>db:</strong> this is where the section for the db setup begins.
<br>
<strong>image:</strong> the image from the Docker repository to download
<br>
<strong>volumes:</strong> a mapping of the local directories (on the left of the colon) to the directories used by the container (on the right of the colon).
<strong>- ./db/pgdata:/pgdata</strong> the local directory at ./db/pgdata will be mapped to /pgdata of the postgres. If the directory doesn't exist then it creates it automatically.
<strong>- ./db/sample-setup.sql:/docker-entrypoint-initdb.d/sample-setup.sql</strong> the sample-setup.sql will be the entrypoint when the docker initialises and creates the schema if it doesn't exist.
ports: the ports used by postgres.
<strong>- "5432:5432"</strong> Here we map the default port 5432 of postgres' container to the 5432 port of the host (host is on the left of the colon, container on the right).
<strong>environment:</strong> In this section we set the postgres environmental variables.
<strong>- POSTGRES_USER=docker</strong> the docker user
<strong>- POSTGRES_PASSWORD=docker</strong> user's password
<strong>- PGDATA=/pgdata</strong> the pgdata directory
<br>
In this phase, we can run the docker-compose file and load our database:<br>
<pre><strong>$services&gt;</strong>docker-compose up</pre>
The first time that docker-compose will run, it will download image for postgres, next it will create and finally run the container. In the first run it will create the directory pgdata and it will read and execute the sample-setup.sql to create the database. Then you can connect to the database with a client that you want and populate the database with data. To stop the container, just give Ctrl-C.
<br>
All local directories in the mappings are referred to the location of the docker-compose file.
<br>
The database will hold the data no matter how many times you will start and stop the postgres container, as long as you don't change the position of the docker-compose file or the db/pgdata directory.
<br>
&nbsp;
<br>
<strong>Application Server setup</strong>
<br>
Next, we will add the application server in the docker-compose file. As mentioned in the beginning, we will use the glassfish4 application server, who's image can be found in the docker repository.
<br>
For a simple example we would just need a war file to deploy. But in this tutorial, since we already have setup the database in the docker-compose file, we will show how to connect the glassfish container with the postgres container.
<br>
First thing to do is to copy the sample.war file to the <strong>ws</strong> directory. Next, for letting glassfish to connect to postgres, the postgres driver is needed to be placed in the lib directory. So, we copy the jar file of the postgres driver postgresql-X.X.XXXX.jar at the ws directory. These two files (war and jar) will be mapped to the proper locations within the container at the docker-compose file.
<br>
So, open again the docker-compose file and add under the last line the following:
<pre>  ws:
    image: oracle/glassfish
    volumes:
      - ./ws/postgresql-9.4.1208.jre6.jar:/glassfish4/glassfish/domains/domain1/lib/ext/postgresql-9.4.1208.jre6.jar
      - ./ws/sample.war:/glassfish4/glassfish/domains/domain1/autodeploy/sample.war
    ports:
      - "4848:4848"
      - "8080:8080"
      - "8181:8181"
    links:
      - db
</pre>
Don't forget that since <strong>ws</strong> section is under <strong>services</strong>, like <strong>db</strong>, it should be indented by two spaces as well.
<br>
<strong>image</strong> describes the docker image of glassfish
<br>
<strong>volumes</strong> map local directories and files (left of the colon) to the container's directories and files (right to the colon):
<strong>- ./ws/postgresql-9.4.1208.jre6.jar:/glassfish4/glassfish/domains/domain1/lib/ext/postgresql-9.4.1208.jre6.jar</strong> maps the postgres driver to the proper file and location
<strong>- ./ws/sample.war:/glassfish4/glassfish/domains/domain1/autodeploy/sample.war</strong> maps the war file to be deployed in the autodeploy directory of the container.
<strong>ports</strong> are the ports used by the glassfish container (right member) mapped to the local host's ports (left member)
<strong>- "4848:4848"</strong> administrator's page port
<strong>- "8080:8080"</strong> glassfish port
<strong>- "8181:8181"</strong> secure port
<strong>links</strong> links the configuration of the ws to the configuration of the db.
<br>
Previously we had only one configuration (db) and the command <strong>docker-compose up</strong> would run only the db container. Now that we added the ws configuration, the same command will run <strong>both</strong> containers at the same time.
<br>
In case we need to run <strong>only the db</strong> container run the command:
<pre><strong>$services&gt;</strong>docker-compose run db</pre>
Since the ws configuration has link to the db configuration, it will complain if the db is not running.
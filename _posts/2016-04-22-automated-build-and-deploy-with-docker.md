---
layout:     post
title:      "Automated Building and Deployment with Docker"
date:       2016-04-22 14:43:00
categories: devops
redirect_from: "/automated-building-deployment-docker-microservices"
---

In this guide I will focus on automatic deployments using Spring Boot and Docker. My intuition was to build and deploy services developed with Spring Boot seamlessly without using a full-fledged CI tool. A simple bash script automates the process of getting source code, building, dockerizing, deploying and running. My requirement was that these operations must be done as easy as clicking an URL.

## The Service

The service is assumed to be developed with [Spring Boot](http://projects.spring.io/spring-boot/) and optionally uses a database for storing data. Spring Boot's special maven plugin allows running the project in a well-configured embedded container like Tomcat or Jetty. The maven plugin produces a runnable `jar` with embedded container by `mvn package` command.

Note that using Spring Boot is optional: you may use vanilla Java EE configuration and use a different `Dockerfile` that includes a container (see docker containers for Tomcat), however I will only focus on the Spring Boot case for now.

You can use any database for storing data. In the last section, we will deploy MongoDB or MySQL as docker containers and how to link to them from your service container. Deployed database containers can also be shared among multiple services and backing up the data becomes trivial.

## Dockerizing your Maven Project

Add a new file `src/main/docker/Dockerfile`. I am assuming your maven artifactId is 'myapp' and 'myapp.war' (Also works with `jar`) is the name of your packaged runnable jar under 'target' directory. Make sure it runs with `java -jar myapp.war`.

    FROM frolvlad/alpine-oraclejdk8:slim
    VOLUME /tmp
    EXPOSE 8080
    ADD myapp.war app.jar
    RUN sh -c 'touch /app.jar'
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

You don't need to know how to write a `Dockerfile`, but this is a fairly simple example.

*   Every `Dockerfile` extends from another, as you can see this extends from [`frolvlad/alpine-oraclejdk8`](https://hub.docker.com/r/frolvlad/alpine-oraclejdk8/) so that our container comes with Oracle Java 8 preinstalled.
*   We expose port `8080` to outside so that our container's services can be accessed from this port. The port number we give here is trivial, it will not affect the output and we will map it to a different port later on. However if you use `server.port` configuration parameter of Spring Boot, you need to `EXPOSE` that port. I assume your service is a web application hosted on a single port.
*   Include myapp.war as app.jar in the container we will build.
*   We state run java program and conclude the file. You may add any Java VM parameters to the last line (such as `-Xmx1G`).

We will be using docker maven plugin of spotify. Add below part to your `pom.xml`. Note that we put `localhost:5000` as group name. This decision allows you to change the build server later on without affecting your project files.

    <build>
      <finalName>myapp</finalName>
      <plugins>
        <plugin>
           <groupId>com.spotify</groupId>
           <artifactId>docker-maven-plugin</artifactId>
           <version>0.2.3</version>
           <configuration>
              <imageName>localhost:5000/${project.artifactId}</imageName>
              <dockerDirectory>src/main/docker</dockerDirectory>
              <imageTags>
                 <imageTag>latest</imageTag>
              </imageTags>
              <resources>
                 <resource>
                    <targetPath>/</targetPath>
                    <directory>${project.build.directory}</directory>
                     <include>${project.build.finalName}.${project.packaging}</include>
                 </resource>
              </resources>
           </configuration>
        </plugin>
      </plugins>
    </build>

Run `mvn package docker:build` (you have to have a docker installation on your development machine) to build docker image and install it on your machine's library. You can see your images by typing `docker images`.

You have successfully dockerized your project!

Run `docker run -p 8080:8080 -i --name myapp localhost:5000/myapp` to run a container from the image built. You will see the output of the program. Exit with `ctrl+c` and remove the container with `docker rm -vf myapp`.

Note that your database connections will not work and you may see exceptions: we will connect a database later on. We will not use docker while developing the software so our development environment does not need to have a full-fledged docker infrastructure with database containers. Ideally, your dev environment does not even need to have docker installed, we've just used this testing step (`mvn docker:build`) to see our configuration is fine. If you don't want to install docker on your dev environment you can also work on the build machine later on.

## The builder: build.sh

I recommend sparing a linux machine for building the service and running docker containers. You may also just use your local machine for everything, or two machines for building and running. This architecture requires no dependency or setup on developer machines.

`build.sh` is a simple bash script to pull your project from `git` and build it. You may also use `svn` or any other version control system. This script creates a new directory for every build and timestamps them but you may just change this part to your liking.

    #!/bin/bash
    # author: Selim Eren Bekçe
    # stop on any error
    set -e
    # app is the first parameter
    app="$1"
    echo "app: $app"
    NOW=$(date +"%Y%m%d-%H%M%S")
    mkdir -p "~/build/$app-$NOW"
    cd "~/build/$app-$NOW"
    # it should be no password login. use ssh keys if necessary.
    git clone --depth=1 "ssh://git@myserver/$app.git"
    cd "$app"
    # build app
    mvn -DskipTests=true package docker:build
    # docker image is ready

    chmod +x build.sh

Try running `./build.sh myapp` and see that it clones your project from git and packages it. Then, try running `curl http://server:8082/?app=myapp` or just go to that url on your browser and see the output as your project is getting built.

## Docker configuration

This part will explain how we will use docker containers with this setup. Some familiarity with `docker` is assumed. You only need to know what a container is, how to start a container, how to list images and container and how to remove them.

We have built our docker image in previous step via the maven plugin. We will now add startup logic to `build.sh`:

    # docker image is ready
    # stop current container
    docker stop $app || true
    # remove current container
    docker rm -v $app || true
    # remove current image
    docker rmi localhost:5000/$app:current || true
    # tag latest image as current
    docker tag localhost:5000/$app:latest localhost:5000/$app:current
    # run the container
    docker run -P -d --name $app --link mongo -e VIRTUAL_HOST=$app.$(hostname -f) localhost:5000/$app:latest
    # clean up
    cd ~
    rm -rf ~/build/$app-$NOW

These commands are responsible to restart the container. It also does remove previous versions of your docker container from the local image store. Without this, your images will keep growing until your disk storage becomes full.

You may change this part to your liking. For my projects, I add `if [ "$app" = "myapp" ]; then; docker run ...; fi` clause for each project so that I have full control over which command gets executed for each project, since each project may have couple of special configuration parameter.

## Reverse proxy with nginx

Your containers will provide some web service and you'd want to access it. Normally, each service will be associated with a different port number and you can easily access it with `http://server:port`. However, there is a more elegant solution, which uses http reverse proxy mechanism so that all services will be provided from a single port and the routing decision will be performed by looking at the common name (dns name) of the incoming request.

An example routing could be like below. Assume each app is a docker container.

app1.server.example.com:80 -> server.example.com:8091 (app1)  
app2.server.example.com:80 -> server.example.com:8092 (app2)  
app3.server.example.com:80 -> server.example.com:8093 (app3)

In this example there are 3 apps served under different port numbers (8091, 8092, 8093). Normally you have to remember the port names to access them. However a reverse proxy server sits on a single port and routes requests to corresponding servers via DNS matching. This first part is the same principle used for multiple (virtual) hosts on a single web server. The second (routing) part is the reverse proxy mechanism.

Nginx is commonly used as a reverse proxy mechanism in the industry. There is a docker container for nginx exactly tailored for providing proxy for your containers. Run it with:

    docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro --name nginx-proxy jwilder/nginx-proxy`

Make sure your 80 port is unused (this machine is just for building and running containers right?) or just select a different port by: `-p 81:80`

You must also make sure your client wildcard DNS records point to this server. `app.server.example.com` and `appX.server.example.com` and `server.example.com` must all point to same ip address. You can also choose to use`appX.example.com`, your call. If you just want to test it, edit `/etc/hosts` file on client's machine.

In the previous subsection, we added `-e VIRTUAL_HOST=$app.$(hostname -f)` for running the container. This env variable is watched by the `nginx-proxy` we created earlier and it automatically configures itself to provide reverse proxy for your newly created container. See reference [here](https://github.com/jwilder/nginx-proxy).

Note that `$(hostname -f)` expands into `server.example.com` (fully qualified name of your server, make sure it is correct and matches with DNS records). You can also use a static value, if you like.

Now, after starting `nginx-proxy`, run `build.sh myapp` again (make sure to include the commands in previous step) and if all goes well just go to `app.server.example.com` to see your running application!

## Database Containers

A web service usually depends on a database server. In this post I will explain how to use mongodb and mysql as a docker container by linking them from your service (app) container.

#### MongoDB

Run mongo container ( no need to expose port as we will use [docker links](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/) )

    docker run --name mongo -d mongo

Link an app (change `build.sh`):

    docker run --link mongo ...

This will put `mongo` entry to `hosts` file of container machine so that it can access the database via this hostname. So you have to change database url on your project. Ideally, you should have multiple profiles, as in Spring, you can create `application-prod.properties` file and put relevant configuration property (such as `spring.data.mongodb.uri=mongodb://mongo/myapp`) to this file and activate `prod` profile during runtime. [See this](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-properties-and-configuration.html#howto-set-active-spring-profiles) to see how to activate profiles.

#### MySQL

Run MySQL Server: (sets root password, creates a single database and a user)

    docker run --name mysql -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=mydb -e MYSQL_USER=myuser -e MYSQL_PASSWORD=mypassword -d mysql:5.6

MySQL Client: (connects to previously started server container)

    docker run -it --link mysql:mysql --rm mysql:5.6 sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'

Link an app (change `build.sh`):

    docker run --link mysql ...

Same configuration change: point your jdbcUrl to `mysql` host.

## Node.js script (optional)

My requirement was to initiate a build using my web browser. So I have installed `node.js` on this machine to start a simple web service to start the build process in a remote location. I have written a simple node.js script to serve under a port and kickoff the build script and relay the process output to the caller so that one can see the log of the build and deploy progress.

```js
// node.js script to start build process
// author: Selim Eren Bekçe

const http = require('http');
const url = require('url');
const sys = require('util');
const exec = require('child_process').exec;
const spawn = require('child_process').spawn;

function handleRequest(request, response){
    response.setHeader('Transfer-Encoding', 'chunked');
    console.log('Received request. Url: ' + request.url);
    var query = url.parse(request.url, true).query;
    var app = query.app;
    if(app == undefined){
        response.writeHead(400, 'Bad request');
        response.end("No 'app' parameter");
        return;
    }
    console.log('App: '+app);
    var child = spawn('/home/division/build.sh', [app]);
    child.stdout.on('data', function (data) {
        process.stdout.write(data.toString());
        response.write(data);
    });
    child.stderr.on('data', function (data) {
        process.stdout.write(data.toString());
        response.write(data);
    });
    child.on('close', function (code) {
        response.end('Finished. Exit code: '+code);
        console.log('child process exited with code ' + code);
    });
}
var server = http.createServer(handleRequest);
server.listen(PORT, function(){
    console.log("Server listening on: http://localhost:%s", PORT);
});
```

The `app` parameter is used so that same infrastructure can be used to build multiple projects. It is relayed to `build.sh` we built earlier. It relays its output directly to the http client in chunked transfer encoding so that the client can see immediate results on the screen to track the build progress. It automatically stops the stream when the process is complete and prints the process exit code.

Run this with `node listen.js` and send a request with `curl http://server:8082/?app=myapp` to build your project!

**Notes**: Note that if this server is public, it may be logical to add an authentication to this endpoint or only allow some specific clients to access it. This is fairly easy in node.js. Use [http-auth](https://www.npmjs.com/package/http-auth) package to require authentication. Also see [http](https://nodejs.org/api/http.html) for more information on the official http plugin. For [node.js](http://node.js/) there is an also more advanced [express.js](http://express.js/) framework, but I have consciously not used that to be simple as possible.

This node.js implementation is very trivial and easy to grasp & use. You may also write a simple python script for this task or any other lightweight framework of your choice. You may also skip this part completely.

## Registry (optional)

This part is optional in `build.sh`, it adds the support to push the image to a `docker-registry` server so that it can be usable for more people. It is required for distributed setup where builder and docker host machines are different so that builder machine can directly push the built docker image to the remote repository for the docker host to run it.

Run registry via:

    docker run -d --name registry -p 5000:5000 registry:2

Add to `build.sh`:

    # optional: push this build to a docker-registry
    docker push localhost:5000/$app:latest

## Conclusion

That's all! That was a long post. If you have any questions, please don't hesitate to ask.

[See this gist for full code/script samples](https://gist.github.com/bekce/4c5a4c6af8c5fdb8cfefe166bc209649).

Good luck!

# Docker & DevOps // Tutorial 1

## Prerecquisite
For the following tutorial, you gonna need [Docker](https://www.docker.com/) installed on your machine, [Git](https://github.com) and an IDE (Please respect yourself) or at least a text editor like [Vim](https://www.vim.org/) or [the other one which we do not care about](https://www.gnu.org/software/emacs/). 
You do not need to be on Linux, Windows or Mac, thanks to docker :)

You should fork now the project [https://github.com/adrienestimeo/docker-tutorial](https://github.com/adrienestimeo/docker-tutorial), we will work on it in the part 2.


## Docker

### Introduction

> "*Build, Ship, and Run Any App, Anywhere*"

Docker lets you deploy application in custom environment via containers. *Docker provides an additional layer of abstraction and automation of operating-system-level virtualization on Windows and Linux* (*[Wikipedia](https://en.wikipedia.org/wiki/Docker_(software))*). 

To keep it simple and in a *non-computer-science* language: **Docker** lets you deploy your application in any system by using your configuration in order to create an expected environment.

### Install

The installation part is probably the worst part of Docker (especially on Windows). Depending on your OS / version / build and finally configuration, you might have a different installation but also different commands than other people. 

Go on **[Docker](https://www.docker.com/)** and follow the instructions.

To test your installation, try the next command:
```
docker run hello-world
```

If everything is fine, **Docker** we let you know :)

> In order to use Docker, it's highly recommended to use a *bash shell*. 
> If you are on Windows and if you had to install the **[Docker Toolbox](https://docs.docker.com/toolbox/toolbox_install_windows/)**, you can use the ***Docker Quickstart Terminal***, if not, try to use ***Powershell*** or ***Git Bash***.

### Docker Project Architecture

Depending on the complexity of your project, you can have two different architecture.

1. **Without** *docker-compose* (simple app)
```
$ 	 MyProject/
		| .git
		| .gitignore
		| Dockerfile
		| public/
		    | index.html
		    | ...
		| server.js
		| ...
```

2. **With** *docker-compose* (complex app)
```
$ 	 MyProject/
		| docker-compose.yml
		| Database/
			| .git
			| .gitignore
			| Dockerfile
			| db.conf
			| ...
		| Logic/
			| .git
			| .gitignore
			| Dockerfile
			| server.js
			| ...
		| API/
			| .git
			| .gitignore
			| Dockerfile
			| api.js
			| ...
		| Reverse_Proxy/
			| .git
			| .gitignore
			| Dockerfile
			| nginx.conf
			| ...
```

### Dockerfile

The ***Dockerfile*** is the configuration file of your container. You specify all the environment needed by your application.
Whenever it's possible, use current *Official Repositories* as the basis.

```Dockerfile
# The minimum environment (Node, php, etc...)
FROM <image>:<tag>

# Use RUN to execute classic command as mkdir, cd, etc...
RUN <cmd>

# Use EXPOSE to open a port to the Docker machine, for example 80, 4242, etc...
EXPOSE <port>

# The ENTRYPOINT & CMD let you tell to Docker what to do when the container is mounted
ENTRYPOINT [ "command_or_file_to_execute" ]
CMD [ "arg_1", "arg_2", "arg_3"]
```

> You should check the documentation, we can do almost everything in a Dockerfile !

### Docker compose

***docker-compose*** lets you build and run multiple containers as you can do with a script. The goal is to clean how you organize your application(s), and use a "configuration" file instead of script. Check the following example showing how a project which uses `nginx` (Reverse proxy), `node` (Server logic) and `mongo` (Database) is built:

```dockerfile

# Specify version of docker-compose
version: "3.2"

# Create specific networks to use
networks:

    # An open network, which will be open to the world
    open-network:
        driver: bridge
        
    # An internal network, which will be open to specific microservice
    internal-network:
        driver: bridge
        
# List our services
services:

    # A reverse proxy
    ms-reverse-proxy:
    
        # Use remote image
        image: traefik:1.7.16
        
        # Always restart (in case of crash)
        restart: always
        
        # Network to use
        # | open-network ONLY
        networks:
            - open-network
            
        # The argument when launching the application
        command: --docker
        
        # Specify labels, used by the application
        labels:
            - "traefik.frontend.rule=Host:monitor.estimeo.localhost"
            
        # Host data which the container will have access to
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            
        # Port used by the application and linked to the host
        # | Port 80 of the container linked to the port 80 of the host
        ports:
            - "80:80"
            
    # A frontend service
    ms-front-website:
    
        # Use local image
        build: ./ms-front-website
        
        # Always restart (in case of crash)
        restart: always
        
        # Network to use
        # | open-network AND internal-network
        networks:
            - open-network
            - internal-network
            
        # This service depends on another one. So it cannot be fully operationnal without it
        # | ms-mongo-database, the database of our architecture
        depends_on:
            - ms-mongo-database
            
        # Specify a file hosting our environment configuration (environment variables)
        env_file: production.env
        
        # Specify labels, used by the application
        labels:
            - "traefik.domain=localhost"
            - "traefik.port=8081"
            - "traefik.frontend.rule=Host:myapp.localhost"
            - "traefik.backend=ms-front-website"
            
    # A worker microservice
    ms-worker-node:
    
        # Use local image
        build: ./ms-worker-node
        
        # Always restart (in case of crash)
        restart: always
        
        # Network to use
        # | internal-network ONLY
        networks:
            - internal-network
            
        # This service depends on another one. So it cannot be fully operationnal without it
        # | ms-mongo-database, the database of our architecture
        depends_on:
            - ms-mongo-database
            
        # Specify a file hosting our environment configuration (environment variables)
        env_file: production.env
        
        # Specify labels, used by the application
        labels:
            - "traefik.domain=localhost"
            - "traefik.backend=ms-worker-node"
            
    # A database
    ms-mongo-database:
    
        # Use remote image
        image: "mongo:3.4.23"
        
        # Always restart (in case of crash)
        restart: always
        
        # Network to use
        # | internal-network ONLY
        networks:
            - internal-network
            
        # Specify a file hosting our environment configuration (environment variables)
        env_file: production.env
        
        # Specify labels, used by the application
        labels:
            - "traefik.enable=false"
            
        # Port used by the application and NOT linked to the host
        # | Port 27017 of the container opened on the internal-network ONLY
        ports:
            - "27017"
```

### Docker machine
***docker-machine*** is a service like process. It is used to create and manage all of your containers / images. It differs if you are on *Windows* or *Linux / Unix*:

* *Windows*
```
docker-machine stop
docker-machine start
docker-machine restart
```
* *Linux / Unix*
```
(sudo) (systemctl) stop docker
(sudo) (systemctl) start docker
(sudo) (systemctl) restart docker
```

### Basics command

#### Build / Run

To create and run a project inside a container, we first have to build one:
```bash
docker build -t username/projectname /path/to/dockerfile
```

and finally run it:
```bash
docker run username/projectname
```

or via *docker-compose* (build + run):
```bash
docker-compose up --build
```

#### List / Stop / Remove
```bash
#List
docker ps -a

#Stop
docker stop imagename

#Remove
docker rm imagename
```
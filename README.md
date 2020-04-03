# Docker & DevOps // Tutorial 1

## Prerecquisite
For the following tutorial, you gonna need [Docker](https://www.docker.com/) installed on your machine, [Git](https://github.com) and an IDE (Please respect yourself) or at least a text editor like [Vim](https://www.vim.org/) or [the other one which we do not care about](https://www.gnu.org/software/emacs/). 
You do not need to be on Linux, Windows or Mac, thanks to docker :)

You should fork now the project [https://github.com/adrienestimeo/ms-main-node](https://github.com/adrienestimeo/ms-main-node), we will work on it in the part 2.

For the following tutorial, the architecture will be:


```
| myProject/
    | ms-main-node/
        | .git/
        | app/
            | ...
        | routes/
            | ...
        | gulpfile.js
        | package.json
        | server.js
        | ...
```

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

## Tutorial part 1

### Overview

With the following steps, we will try to transform a monolithic project into an appropriate project using a DevOps approach.

Currently, this is how the project is launched:

1. [Manual]		    We configure our Environment
2. [Manual] 		We create the new feature
3. [Manual] 		We commit / push it (On a ***dev*** branch of course ;) )
5. [Manual] 		We build the environment, and test our project with the new feature
6. [Manual] 		We notify Slack (or whatever) if everything is good (or not).
7. [Manual] 		If everything is good, we pull the new feature on a remote server
8. [Manual] 		We restart your project

### _Dockerized_ App

The first step to improve our current App, is to make it more portable. We saw that Docker can help us on that point.

> Docker alone is not sufficient in a production environment, we can make the system more stable if we also add configuration tool for the environment as well as deployment tool.

In order to _dockerize_ the current app, you will create a **Dockerfile** at the root of the project. This **Dockerfile** will contain (keep the order) :

1. The environment of the container (node in this case) with a specific recent version (alpine)
2. Create the directory in which we will work (**/usr/src/app/**)
3. Specify that we gonna work in this newly created directory
4. Copy the current **package.json** file and install it (`npm install`)
5. Copy the everything in the current folder to the working directory
6. Expose the port in which our server is listening on (hint: check **server.js** file)
7. Specify our Entrypoint `[ "npm", "run", "development" ]`

You will need the following command inside the **`Dockerfile`**:

* `COPY`
* `ENTRYPOINT`
* `EXPOSE`
* `FROM`
* `RUN`
* `WORKDIR`

Try to build and run this container. If you did it well, you should be able to access the website in your favorite browser on the specified `port` on `localhost`.

> Something goes wrong ? Check the following points:
> * Check again your Dockerfile,
> * Check your `build` command,
> * Check your `run` command and the `port` option for example ;),
> * Maybe you cannot access `localhost` directly. In that case, you can check your docker-machine ip with `docker-machine inspect` 

Congratulations, you made your first (or not) *Dockerized*, portable application.

> If you want to start again, do not forget to stop and remove the container, especially if you named it. Otherwise, Docker will throw an error.

### _Migrate to `docker-compose`_

Do not forget that one of our main goal is ___Becoming lazy but efficient___. That's why to write 3 commands to launch and clean our application is clearly too much for us. Let's fix that with `docker-compose`.

This **docker-compose.yml** should do the following:

* Specify a version, at least `3.x`
* Create a network with `bridge` driver
* Create an entry `ms-main-node` which will build from **`ms-main-node`** directory. This entry should also specify the internal port used by the server linked to the same port on the host machine and the network created previously.

You can now test everything with a simple `docker-compose up --build`. If everything is fine, you should be able to access your application in your browser via the same method than the one before

### Adding a Reverse Proxy

In order to create a clean application, we will add a Reverse Proxy: **[Traefik ](https://docs.traefik.io/)** (*Cocorico !*)

We will add the service `ms-reverse-proxy` in the `docker-compose.yml` with the following rules:

* Use the official `traefik` with the version `1.7.16`
* Will always restart if any problem occurs
* Use the network previously created
* The line `command: --docker`
* Use the docker volumes: `- /var/run/docker.sock:/var/run/docker.sock`
* Use port `80` on the docker network and on the machine

You will also have to modify the `ms-main-node` entry by adding the following labels:

* `- "traefik.domain=localhost"`
* `- "traefik.port=8081"`
* `- "traefik.frontend.rule=Host:docker-tutorial.localhost"`
* `- "traefik.backend=ms-main-node"`

Same as above, if everything's fine when launching again your docker-compose, you should be able to access this time [http://docker-tutorial.localhost/](http://docker-tutorial.localhost/) in your browser.

### Create our first worker

Now that our environment is well configured, it's time to create our first microservice worker: `ms-number-computer`. which goal is to compute number.

In order to be a good DevOps developer, we will use a TDD (*Test Driven Development*) approach.
Let's create a new project on github, named `ms-number-computer` and clone it.

The current project architecture should be the following:

```
| myProject/
    | docker-compose.yml
    | ms-main-node/
        | app/
            | ...
        | routes/
            | ...
        | Dockerfile
        | gulpfile.js
        | package.json
        | server.js
        | ...
    | ms-number-computer/
        | ...
```

We can see that the `docker-compose.yml` is above each microservice, and `ms-main-node` and `ms-number-computer` are 2 microservices, with different goal in our project.

We will create the following files in `ms-number-computer`:

1. `README.md` in order to document our micro-service !
2. `package.json` with our node configuration
3. `Dockerfile`, no need to explain why
4. `Dockerfile_test`, same as Dockerfile, but for testing purpose.
5. `config.json` which will contain our microservice messaging system configuration
6. `manager.js` which will manager our microservice messaging system
7. `computer.js` which will contain the main code
8. `computer.spec.js` with the test suit for `computer.js`

Following are the draft of each file except the `README.md` that you will complete and the `Dockerfile` which is the same as the one of `ms-main-node`:

___

`package.json`:
```json
{
    "name": "ms-number-computer",
    "version": "1.0.0",
    "description": "Main computer",
    "author": "Adrien Fenech <adrien.fenech@estimeo.com>",
    "main": "manager.js",
    "scripts": {
        "development": "node manager.js",
        "test": "mocha **.spec.js"
    },
    "devDependencies": {
        "mocha": "^4.0.1",
        "chai": "^4.1.2"
    },
    "dependencies": {
        "ms-manager": "^0.1.0"
    }
}
```
___

`config.json`:
```json
{  
    "environment": "development",  
    "hydra": {  
        "serviceName": "ms-number-computer",  
        "serviceIP": "",  
        "servicePort": 9000,  
        "serviceType": "computation",  
        "serviceDescription": "Computation of number",  
        "redis": {  
            "url": "redis://ms-redis-cache/0"
        }  
    }  
}
```
___

`manager.js`:
```javascript
const MM = require('ms-manager');  
let config = require(`./config.json`) || {};  

MM.init(config, (err, serviceInfo) => {  
    if (err) {  
        return console.error(err);  
    }
    
    /**  
    * Our micro-service is now up. 
    * */
    console.log('#Micro-service UP#');  
    
    /**
    * You can now subscribe to specific message
    */
    MM.subscribe('add', (bdy, msg) => {  
        /**
        * TODO: Uncomment when operationnal
        **/
        // const computer = require('./computer');
        // try {
        //     const result = computer.add(bdy.a, bdy.b);
        //     msg.reply({ result });
        // } catch (err) {
        //     console.error(err);
        //     return msg.replyErr(err);
        // }
    });
});
```
___

`computer.js`:
```javascript
module.exports = {
    // TODO: Create our computer function here
    add: function(a, b) {
        
        // TODO
        
        return 0;
    }
};
```


___

`computer.spec.js`:
```javascript
'use strict';

const should = require('chai').should();
const expect = require('chai').expect;
const computer = require('./computer');
// TODO: Create our computer function test here

describe('Computer test suit', () => {
    describe('"add" function', () => {
        it('should exist', () => {
            expect(computer.add).to.be.a('function');
        });

        describe('Error', () => {
            it('should throw error if no argument provided', () => {
                expect(computer.add.bind(computer)).to.throw('Arguments missing.');
            });

            it('should throw error if the first argument is null', () => {
                expect(computer.add.bind(computer, null, 2)).to.throw('"null" is not a valid number [arg 0].');
            });

            it('should throw error if the second argument is null', () => {
                expect(computer.add.bind(computer, 40, null)).to.throw('"null" is not a valid number [arg 1].');
            });

            it('should throw error if the first argument is undefined', () => {
                expect(computer.add.bind(computer, undefined, 2)).to.throw('"undefined" is not a valid number [arg 0].');
            });

            it('should throw error if the second argument is undefined', () => {
                expect(computer.add.bind(computer, 40, undefined)).to.throw('"undefined" is not a valid number [arg 1].');
            });

            it('should throw error if the first argument is a String', () => {
                expect(computer.add.bind(computer, "40", 2)).to.throw('a String is not a valid number [arg 0].');
            });

            it('should throw error if the second argument is a String', () => {
                expect(computer.add.bind(computer, 40, "2")).to.throw('a String is not a valid number [arg 1].');
            });
        });

        describe('Compute', () => {
            it('0 + 0 should be equal to 0', () => {
                expect(computer.add(0, 0)).to.be.equal(0);
            });

            it('-1 + 1 should be equal to 0', () => {
                expect(computer.add(-1, 1)).to.be.equal(0);
            });

            it('1 + -1 should be equal to 0', () => {
                expect(computer.add(1, -1)).to.be.equal(0);
            });

            it('40 + 2 should be equal to 42', () => {
                expect(computer.add(40, 2)).to.be.equal(42);
            });

            it('2 + 40 should be equal to 42', () => {
                expect(computer.add(2, 40)).to.be.equal(42);
            });
        });
    });
});
```

___

`Dockerfile_test`
The docker `Dockerfile_test` differs on the entrypoint, which will be `ENTRYPOINT [ "npm", "run", "test" ]`



You can start to test your application with the following commands:
 
1. `docker build -t ms-number-computer --file Dockerfile_test .`
2. `docker run --name ms-number-computer ms-number-computer`
3. `docker stop ms-number-computer`
4. `docker rm ms-number-computer`
 
You should see that the test is failing, because the function `add` is missing in the `computer.js` file. But it also means that our microservice is nicely setup.
In order to do TDD, you will first have to write the test suit. We can start with the current tests and I recommend you to add many, many, MANY more.
 
Time for you to work and write the code of the `add` function. When your test suite will pass, you can continue !

Let's add it to our system. In order to do that, modify our `docker-compose.yml`:
 
1. Create a new network `internal-network`,
2. Create a new service `ms-redis-cache` with the following configuration
    ```
    ms-redis-cache:
        image: "redis:3.2.11-alpine"
        restart: always
        networks:
            - internal-network
        command: redis-server --appendonly yes
        labels:
            - "traefik.enable=false"
        ports:
            - "6379"
    ```
3. Adding the new service `ms-number-computer` which will restart always, build from `./ms-number-computer` use both `internal-network` and `external-network`.
4. Adding the `internal-network` to `ms-main-node`.
5. Add the following lines in both `ms-main-node` and `ms-number-computer`.
    ```
    depends_on:
        - ms-redis-cache
    ```
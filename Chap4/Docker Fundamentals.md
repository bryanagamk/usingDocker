# Docker Fundamentals
In this chapter, we’ll expand on the fundamental Docker concepts. We’ll start by looking at the overall architecture of Docker, including the technologies it builds on. This is followed by more in-depth sections on building Docker images, networking containers, and handling data in volumes. The chapter ends with an overview of the remaining Docker commands.

## The Docker Architecture
In order to understand how best to use Docker and some of the more unusual behavior in Docker, it’s good to have a rough understanding of how the Docker platform is put together under the covers.

* At the center is the Docker daemon, which is responsible for creating, running, and monitoring containers, as well as building and storing images, both of which are represented on the right of the diagram.

*  The Docker client is on the left-hand side and is used to talk to the Docker dae mon via HTTP. By default, this happens over a Unix domain socket, but it can also use a TCP socket to enable remote clients or a file descriptor for systemdmanaged sockets.

* Docker registries store and distribute images. The default registry is the Docker Hub, which hosts thousands of public images as well as curated “official” images. Many organizations run their own registries that can be used to store commercial or sensitive images as well as avoiding the overhead of needing to download images from the Internet.

## Underlying Technologies
The Docker daemon uses an “execution driver” to create containers. By default, this is Docker’s own runc driver, but there is also legacy support for LXC. Runc is very closely tied to the following kernel features:
* cgroups, which are responsible for managing resources used by a container (e.g., CPU and memory usage).

* namespaces are responsible for isolating containers; making sure that a container’s filesystem, hostname, users, networking, and processes are separated from the rest of the system.

Another major technology underlying Docker is the Union File System (UFS), used to store the layers for containers. The UFS is provided by one of several storage drivers, either AUFS, devicemapper, BTRFS, or Overlay.

## Surrounding Technologies
The Docker engine and the Docker Hub do not in-and-of themselves constitute a complete solution for working with containers. 

Most users will find they require supporting services and software, such as cluster management, service-discovery tools, and advanced networking capabilities.

The current list of supporting technologies supplied by Docker includes:
* Swarm
    Docker’s clustering solution. Swarm can group together several Docker hosts, allowing the user to treat them as a unified resource. 
* Compose
    Docker Compose is a tool for building and running applications composed of multiple Docker containers. It is primarily used in development and testing rather than production. See “Automating with Compose” for more details.
* Machine
    Docker Machine installs and configures Docker hosts on local or remote resources. Machine also configures the Docker client, making it easy to swap between environments. See Chapter 9 for an example.
* Kitematic
    Kitematic is a Mac OS and Windows GUI for running and managing Docker containers.
* Docker Trusted Registry
    Docker’s on-premise solution for storing and managing Docker images. Effectively a local version of the Docker Hub that can integrate with an existing security infrastructure and help organizations comply with regulations regarding the storage and security of data.

There is already a large list of services and applications from third parties that build on or work with Docker. Several solutions have already emerged in the following areas:
* Networking
    Creating networks of containers that span hosts is a nontrivial problem that can be solved in a variety of ways. Several solutions have appeared in this area, including Weave and Project Calico.
* Service discovery
    When a Docker container comes up, it needs some way of finding the other services it needs to talk to, which are typically also running in containers.
* Orchestration and cluster management
    In large container deployments, tooling is essential in order to monitor and manage the system. Each new container needs to be placed on a host, monitored, and updated.

## Docker Hosting
Many of the traditional cloud providers, including Amazon, Google, and Digital Ocean, have brought out some level of Docker offering.
Google’s Container Engine may be the most interesting of these, as it is built directly on top of Kubernetes. Of course, even when a cloud provider doesn’t have a specific Docker offering, it’s normally still possible to provision VMs that can run Docker containers.

## How Images Get Built
We saw in “Building Images from Dockerfiles” that the primary way to make new images is through Dockerfiles and the docker build command. This section will look at what happens here in a little more depth and end with a guide to the various instructions that can be used in a Dockerfile.
It’s handy to have some understanding of how the build command works internally, as its behavior can sometimes be surprising.

### The Build Context
The docker build command requires a Dockerfile and a build context (which may be empty). The build context is the set of local files and directories that can be referenced from ADD or COPY instructions in the Dockerfile and is normally specified as a path to a directory.
For example, we used the build command docker build -t test/cowsay-dockerfile . in *“Building Images from Dockerfiles”*, which sets the context to '.' , the current working directory. All the files and directories under the path form the build context and will be sent to the Docker daemon as part of the build process.

In cases where a context is not specified-if only a URL to a Dockerfile is given or the contents of a Dockerfile is piped from STDIN --the build context is considered to be empty.

If a URL beginning with http or https is given, it is assumed to be a direct link to a Dockerfile. This is unlikely to be very useful, as no context is associated with the Dockerfile (and links to archives are not accepted).

A git repository can also be given as the build context. In this situation, the Docker client will clone the repository and any submodules to a temporary directory that is then sent to the Docker daemon as the build context. Docker will interpret the context as a git repository if the path begins with github.com/, _ git@, or _git://. In general, I would suggest avoiding this method and instead checking out repositories by hand, which is more flexible and leaves less chance for confusion.

The Docker client can also take input on STDIN by giving a " - " as an argument in place of the build context. The input can either be a Dockerfile with no context (e.g., docker build - < Dockerfile) or an archive file that constitutes the context and includes a Dockerfile (e.g. docker build - < context.tar.gz). Archive files can be in tar.gz, xz, or bzip2 format.The location of the Dockerfile within the context can be specified with the -f argument (e.g., docker build -f dockerfiles/Dockerfile.debug .). If unspecified, Docker will look for a file called Dockerfile at the root of the context.

### Image Layers
New Docker users are often thrown by the way images are built up. Each instruction in a Dockerfile results in a new image layer, which can also be used to start a container. The new layer is created by starting a container using the image of the previous layer, executing the Dockerfile instruction and saving a new image. 

When a Dockerfile instruction successfully completes, the intermediate container will be deleted, unless the --rm=false argument was given.

### Caching
Docker also caches each layer in order to speed up the building of images. This caching is very important for efficient workflows, but is somewhat naive. The cache is used for an instruction if:

* The previous instruction was found in the cache and
* There is a layer in the cache that has exactly the same instruction and parent layer (even spurious spaces will invalidate the cache).
Also, in the case of COPY and ADD instructions, the cache will be invalidated if the checksum or metadata for any of the files has changed.

This means that RUN instructions that are not guaranteed to have the same result across multiple invocations will still be cached. Be particularly aware of this if you download files, run apt-get update , or clone source repositories.

### Base Images
When creating your own images, you will need to decide which base image to start from. There are a lot of choices, and it’s worth taking the time to understand the various advantages and disadvantages of each.
The best-case scenario is that you don’t need to create an image at all—you can just use an existing one and mount your configuration files and/or data into it. This is likely to be the case for common application software, such as databases and web servers, where there are official images available.
In general, you are much better off using an official image than rolling your own—you get the benefit of other people’s work and experience in figuring out how best to run the software inside a container. If there is a particular reason an official image doesn’t work for you, consider opening an issue on the parent project, as it is likely others are facing similar problems or know of workarounds.
If you need an image to host your own application, first have a look to see if there is an official base image for the language or framework you are using (e.g., Go or Ruby on Rails). Often you can use separate images for building and distributing your software (e.g., you could use the java:jdk image to build a Java application but then distribute the resulting JAR file using the smaller java:jre image, which gets rid of the unnecessary build tooling). Similarly, some official images (such as node ) have special “slim” builds that remove a lot of development tools and headers.

### Dockerfile Instructions
This section briefly covers the various instructions available for use in Dockerfiles. It doesn’t go deep into details, partly because things are still changing and likely to quickly get out of date and partly because there is comphrensive and always up-to-date documentation available on the Docker website.

The following instructions are available in Dockerfiles:
* ADD
    Copies files from the build context or remote URLs into the image. If an archive file is added from a local path, it will automatically be unpacked.
* CMD
    Runs the given instruction when the container is started. If an ENTRYPOINT has been defined, the instruction will be interpreted as an argument to the ENTRYPOINT (in this case, make sure you use the exec format). 
* COPY
    Used to copy files from the build context into the image. It has two forms, COPY src dest_ and COPY ["src", "dest"] , both of which copy the file or directory at src in the build context to dest inside the container.
* ENTRYPOINT
    Sets an executable (and default arguments) to be run when the container starts. Any CMD instructions or arguments to docker run after the image name will be passed as parameters to the executable.
* ENV
    Sets environment variables inside the image. These can be referred to in subsequent instructions. For example:
```dockerfile
    ...
    ENV MY_VERSION 1.3
    RUN apt-get install -y mypackage=$MY_VERSION
    ...
```

The variables will also be available inside the image.
* EXPOSE
    Indicates to Docker that the container will have a process listening on the given port or ports
* FROM
    Sets the base image for the Dockerfile; subsequent instructions build on top of this image. The base image is specified as `IMAGE:TAG (e.g., debian:wheezy )`
* MAINTAINER
    Sets the “Author” metadata on the image to the given string. You can retrieve this with `docker inspect -f {{.Author}} IMAGE`.
* ONBUILD
    Specifies an instruction to be executed later, when the image is used as the base layer to another image.
* RUN
    Runs the given instruction inside the container and commits the result.
* USER
    Sets the user (by name or UID) to use in any subsequent RUN , CMD , or ENTRYPOINT instructions.
* VOLUME
    Declares the specified file or directory to be a volume. If the file or directory already exists in the image, it will copied into the volume when the container is started. If multiple arguments are given, they are interpreted as multiple volume statements
* WORKDIR
    Sets the working directory for any subsequent RUN , CMD , ENTRYPOINT , ADD , or COPY instructions. Can be used multiple times. Relative paths may be used and are resolved relative to the previous WORKDIR 

### Connecting Containers to the World
Say you’re running a web server inside a container. How do you provide the outside world with access? The answer is to “publish” ports with the -p or -P commands. This command forwards ports on the host to the container. For example:

```
    **$ docker run -d -p 8000:80 nginx**
    af9038e18360002ef3f3658f16094dadd4928c4b3e88e347c9a746b131db5444
    **$ curl localhost:8000**
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
```

The -p 8000:80 argument has told Docker to forward port 8000 on the host to port 80 in the container. Alternatively, the -P argument can be used to tell Docker to automatically select a free port to forward to on the host. For example:

```
    **$ ID=$(docker run -d -P nginx)**
    **$ docker port $ID 80**
    0.0.0.0:32771
    **$ curl localhost:32771**
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
```

The primary advantage of the -P command is that you are no longer responsible for keeping track of allocated ports, which becomes important if you have several containers publishing ports. In these cases you can use the docker port command to discover the port allocated by Docker.

### Linking Containers
Docker links are the simplest way to allow containers on the same host to talk to each other. When using the default Docker networking model, communication between containers will be over an internal Docker network, meaning communications are not exposed to the host network.

Links are initialized by giving the argument --link CONTAINER:ALIAS to docker run , where CONTAINER is the name of the link container 3 and ALIAS is a local name used inside the master container to refer to the link container.
Using Docker links will also add the alias and the link container ID to /etc/hosts on the master container, allowing the link container to be addressed by name from the master container. In addition, Docker will set a bunch of environment variables inside the master container that are designed to make it easy to talk to the link container. For example, if we create and link to a Redis container:

```
    **$ docker run -d --name myredis redis**
    c9148dee046a6fefac48806cd8ec0ce85492b71f25e97aae9a1a75027b1c8423
    **$ docker run --link myredis:redis debian env**
    ATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    HOSTNAME=f015d58d53b5
    REDIS_PORT=tcp://172.17.0.22:6379
    REDIS_PORT_6379_TCP=tcp://172.17.0.22:6379
    REDIS_PORT_6379_TCP_ADDR=172.17.0.22
    REDIS_PORT_6379_TCP_PORT=6379
    REDIS_PORT_6379_TCP_PROTO=tcp
    REDIS_NAME=/distracted_rosalind/redis
    REDIS_ENV_REDIS_VERSION=3.0.3
    REDIS_ENV_REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-3.0.3.tar.gz
    REDIS_ENV_REDIS_DOWNLOAD_SHA1=0e2d7707327986ae652df717059354b358b83358
    HOME=/root
```

we can see that Docker has set up environment variables prefixed with REDIS_PORT , that contain information on how to connect to the container. Most of these seem somewhat redundant, as the information in the value is already contained in the variable name. Nevertheless, they are useful as a form of documentation if nothing else.

By default, containers will be able to talk to each other whether or not they have been explicitly linked. If you want to prevent containers that haven’t been linked from communicating, use the arguments --icc=false and --iptables when starting the Docker daemon. Now when containers are linked, Docker will set up Iptables rules to allow the containers to communicate on any ports that have been declared as exposed

Unfortunately, Docker links as they stand have several shortcomings. Perhaps most significantly, they are static—although links should survive container restarts, they aren’t updated if the linked container replaced. Also, the link container must be started before the master container, meaning you can’t have bidirectional links.

### Managing Data with Volumes and Data Containers
To recap, Docker volumes are directories *(Technically, directories or files, as a volume may be a single file.)* that are not part of the container’s UFS they are just normal directories on the host that are bind mounted (see Bind Mounting) into the container. There are three *(OK, two-and-a-half, depending on how you want to count.)* different ways to initialize volumes, and it’s important to understand the differences between the methods. First, we can declare a volume at runtime with the -v flag:

```
    **$ docker run -it --name container-test -h CONTAINER -v /data debian /bin/bash**
    root@CONTAINER:/# ls /data
    root@CONTAINER:/#
```

This will make the directory /data inside the container into a volume. Any files the image held inside the /data directory will be copied into the volume. We can find out where the volume lives on the host by running docker inspect on the host from a new shell:

```
    **$ docker inspect -f {{.Mounts}} container-test**
    [{5cad... /mnt/sda1/var/lib/docker/volumes/5cad.../_data /data local true}]
```

In this case, the volume /data/ in the container is simply a link to the directory /var/lib/docker/volumes/5cad…/_data on the host. To prove this, we can add a file into the directory on the host:

`   **$ sudo touch /var/lib/docker/volumes/5cad.../_data/test-file**`

And you should immediately be able to see from inside the container:

```
    **$ root@CONTAINER:/# ls /data**
    test-file
```

The second way to set up a volume is by using the VOLUME instruction in a Dockerfile:
```dockerfile
    FROM debian:wheezy
    VOLUME /data
```

The third way is to extend the -v argument to docker run with an explicit directory to bind to on the host using the format -v HOST_DIR:CONTAINER_DIR . This can’t be done from a Dockerfile (it would be nonportable and a security risk). For example:

`   **$ docker run -v /home/bro/data:/data debian ls /data**`

This will mount the directory /home/adrian/data on the host as /data inside the container. Any files already existing in the /home/bro/data directory will be available inside the container. If the /data directory already exists in the container, its contents will be hidden by the volume. Unlike the other invocations, no files from the image will be copied into the volume, and the volume won’t be deleted by Docker (i.e., docker rm -v will not remove a volume that is mounted at a user-chosen directory).

### Sharing Data
The -v HOST_DIR:CONTAINER_DIR syntax is very useful for sharing files between the host and one or more containers. For example, configuration files can be kept on the host and mounted into containers built from generic images. We can also share data between containers by using the --volumes-from CONTAINER argument with docker run . For example, we can create a new container that has access to the volumes from the container in our previous example like so:

```
    **$ docker run -it -h NEWCONTAINER --volumes-from container-test debian /bin/bash**
    root@NEWCONTAINER:/# ls /data
    test-file
    root@NEWCONTAINER:/#
```

It’s important to note that this works whether or not the container holding the volumes ( container-test in this case) is currently running. As long as at least one existing container links to a volume, it won’t be deleted.

### Data Containers
A common practice is to create data containers—containers whose sole purpose is to share data between other containers. The main benefit of this approach is that it provides a handy namespace for volumes that can be easily loaded using the --volumes-from command. For example, we can create a data container for a PostgreSQL database with the following command:

`   **$ docker run --name dbdata postgres echo "Data-only container for postgres"**`

This will create a container from the postgres image and initialize any volumes defined in the image before running the echo command and exiting. There’s no need to leave data containers running, since doing so would just be a waste of resources.
We can then use this volume from other containers with the --volumes-from argument. For example:

`   **$ docker run -d --volumes-from dbdata --name db1 postgres**`

#### Deleting volumes
Volumes are only deleted if:
* The container was deleted with docker rm -v , or
* The --rm flag was provided to docker run
* No existing container links to the volume
* No host directory was specified for the volume (the -v HOST_DIR:CONTAINER_DIR syntax was not used)

At the moment, this means that unless you are very careful about always running your containers like this, you are likely to have orphan files and directories in your Docker installation directory and no easy way of telling what they represent. Docker is working on a top-level “volume” command that will allow you to list, create, inspect, and remove volumes independent of containers

## Common Docker Commands
This section gives a brief (at least in comparison to the official documentation) and nonexhaustive overview of the various Docker commands, focusing on the commands commonly used on a day-to-day basis

### The run Command
We’ve already seen docker run in action; it’s the go-to command for launching new containers. As such, it is by far the most complex command and supports a large list of potential arguments. The arguments allow users to configure how the image is run, override Dockerfile settings, configure networking, and set privileges and resources for the container. The following options control the lifecycle of the container and its basic mode of operation:

-a, --attach
    Attaches the given stream ( STDOUT , etc.) to the terminal. If unspecified, both stdout and stderr are attached. If unspecified and the container is started in interactive mode ( -i ), stdin is also attached. Incompatible with -d 

-d, --detach
    Runs the container in “detached” mode. The command will run the container in the background and return the container ID.

-i, --interactive
    Keeps stdin open (even when it’s not attached). Generally used with -t to start an interactive container session. For example:

    $ docker run -it debian /bin/bash
    root@bd0f26f928bb:/# ls
    ...snip...

--restart
    Configures when Docker will attempt to restart an exited container. The argument no will never attempt to restart a container, and always will always try to restart, regardless of exit status. The on-failure argument will attempt to restart containers that exit with a nonzero status and can take an optional argument specifying the number of times to attempt to restart before giving up (if not specified, it will retry forever). For example, docker run --restart on-failure:10 postgres will launch the postgres container and attempt to restart it 10 times if it exits with a nonzero code.

--rm
    Automatically removes the container when it exits. Cannot be used with -d .

-t, --tty
    Allocates a pseudo-TTY. Normally used with -i to start an interactive container. 
    
The following options allow setting of container names and variables:

-e, --env
    Sets environment variables inside the container. For example:
        $ docker run -e var1=val -e var2="val 2" debian env
        PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        HOSTNAME=b15f833d65d8
        var1=val
        var2=val 2
        HOME=/root
    Also note the --env-file option for passing variables in via a file.

-h, --hostname
    Sets the container’s unix host name to NAME . For example:
    $ docker run -h "myhost" debian hostname
    myhost
    --name NAME
    Assigns the name NAME to the container. The name can then be used to address the container in other Docker commands.

The following options allow the user to set up volumes:

-v, --volume
    There are two forms of the argument to set up a volume (a file or directory within a container that is part of the native host filesystem, not the container’s union file system). The first form only specifies the directory within the container and will bind to a host directory of Docker’s choosing. The second form specifies the host directory to bind to.

--volumes-from
    Mounts volumes from the specified container. Often used in association with data containers 

There are several options affecting networking. The basic commands you can expect
to commonly use are:

--expose
    Equivalent of Dockerfile EXPOSE instruction. Identifies the port or port range as being used in the container but does not open the port. Only really makes sense in association with -P and when linking containers.

--link
    Sets up a private network interface to the specified container.

-p, --publish
    “Publishes” a port on the container, making it accessible from the host. If the host port is not defined, a random high-numbered port will chosen, which can be discovered by using the docker port command. The host interface on which to expose the port may also be specified.

-P, --publish-all
    Publish all exposed ports on the container to the host. A random high-numbered port will be chosen for each exposed port. The docker port command can be used to see the mapping.

The following options directly override Dockerfile settings:

--entrypoint
    Sets the entrypoint for the container to the given argument, overriding any ENTRYPOINT instruction in the Dockerfile.

-u, --user
    Sets the user that commands are run under. May be specified as a username or UID. Overrides USER instruction in Dockerfile.

-w, --workdir
    Sets the working directory in the container to the provided path. Overrides any value in the Dockerfile.

### Managing Containers
In addition to docker run , the following docker commands are used to manage containers during their lifecyle:

docker attach [OPTIONS] CONTAINER
    The attach command allows the user to view or interact with the main process inside the container. For example:
        
        $ ID=$(docker run -d debian sh -c "while true; do echo 'tick'; sleep 1; done;")
        $ docker attach $ID
        tick
        tick
        tick
        tick

    Note that using CTRL-C to quit will end the process and cause the container to exit.

docker create
    Creates a container from an image but does not start it. Takes most of the same arguments as docker run . To start the container, use docker start .

docker cp
    Copies files and directories between a container and the host.

docker exec
    Runs a command inside a container. Can be used to perform maintenance tasks or as a replacement for ssh to log in to a container.
    For example:
        $ ID=$(docker run -d debian sh -c "while true; do sleep 1; done;")
        $ docker exec $ID echo "Hello"
        Hello
        $ docker exec -it $ID /bin/bash
        root@5c6c32041d68:/# ls
        bin dev home lib64 mnt proc run selinux sys usr
        boot etc lib media opt root sbin srv tmp var
        root@5c6c32041d68:/# exit
        exit

docker kill
    Sends a signal to the main process (PID 1) in a container. By default, sends a SIGKILL , which will cause the container to exit immediately. Alternatively, the signal can be specified with the -s argument. The container ID is returned. 
    For example:

        $ ID=$(docker run -d debian bash -c \
        "trap 'echo caught' SIGTRAP; while true; do sleep 1; done;")
        $ docker kill -s SIGTRAP $ID
        e33da73c275b56e734a4bbbefc0b41f6ba84967d09ba08314edd860ebd2da86c
        $ docker logs $ID
        caught
        $ docker kill $ID
        e33da73c275b56e734a4bbbefc0b41f6ba84967d09ba08314edd860ebd2da86c

docker pause
    Suspends all processes inside the given container. The processes do not receive any signal that they are being suspended and consequently cannot shut down or clean up. The processes can be restarted with docker unpause . docker pause uses the Linux cgroups freezer functionality internally. This command contrasts with docker stop , which stops the processes and sends signals observable by the processes.

docker restart
    Restarts one or more containers. Roughly equivalent to calling docker stop followed by docker start on the containers. Takes an optional argument -t that specifies the amount of time to wait for the container to shut down before it is killed with a SIGTERM .

docker rm
    Removes one or more containers. Returns the names or IDs of succesfully deleted containers. By default, docker rm will not remove any volumes. The -f argument can be used to remove running containers, and the -v argument will remove volumes created by the container (as long as they aren’t bind mounted or in use by another container).
    For example, to delete all stopped containers:
        
        $ docker rm $(docker ps -aq)
        b7a4e94253b3
        e33da73c275b
        f47074b60757

docker start
    Starts a stopped container (or containers). Can be used to restart a container that has exited or to start a container that has been created with docker create but never launched.

docker stop
    Stops (but does not remove) one or more containers. After calling docker stop on a container, it will transition to the “exited” state. Takes an optional argument -t which specifies the amount of time to wait for the container to shutdown before it is killed with a SIGTERM .

docker unpause
    Restarts a container previously paused with docker pause

 ### Docker Info
The following subcommands can be used to get more information on the Docker installation and usage:

docker info
    Prints various information on the Docker system and host.

docker help
    Prints usage and help information for the given subcommand. Identical to running a command with the --help flag.

docker version
    Prints Docker version information for client and server as well as the version of Go used in compilation. Container Info The following commands provide more information on running and stopped containers.

docker diff
    Shows changes made to the containers filesystem compared to the image it was launched from. For example:

        $ ID=$(docker run -d debian touch /NEW-FILE)
        $ docker diff $ID
        A /NEW-FILE

docker events
    Prints real-time events from the daemon. Use CTRL-C to quit.

docker inspect
    Provides detailed information on given containers or images. The information includes most configuration information and covers network settings and volume mappings. The command can take one argument, -f , which is used to supply a Go template that can be used to format and filter the output.

docker logs
    Outputs the “logs” for a container. This is simply everything that has been written to STDERR or STDOUT inside the container.

docker port
    Lists the exposed port mappings for the given container. Can optionally be given the internal container port and protocol to look up. Often used after docker run -P <image> to discover the assigned ports.
    For example:

    $ ID=$(docker run -P -d redis)
    $ docker port $ID
    6379/tcp -> 0.0.0.0:32768
    $ docker port $ID 6379
    0.0.0.0:32768
    $ docker port $ID 6379/tcp
    0.0.0.0:32768

docker ps
    Provides high-level information on current containers, such as the name, ID, and status. Takes a lot of different arguments, notably -a for getting all containers, not just running ones. Also note the -q argument, which only returns the container IDs and is very useful as input to other commands such as docker rm .

docker top
    Provides information on the running processes inside a given container. In effect, this command runs the UNIX ps utility on the host and filters for processes in the given container. The command can be given the same arguments the ps utility and defaults to -ef (but be careful to make sure the PID field is still in the output).
    For example:

        $ ID=$(docker run -d redis)
        $ docker top $ID
        UID     PID     PPID    C   STIME   TTY     TIME        CMD
        999     9243    1836    0   15:44   ?       00:00:00    redis-server    *:6379
        $ ps -f -u 999
        UID     PID     PPID    C   STIME   TTY     TIME        CMD
        999     9243    1836    0   15:44   ?       00:00:00    redis-server    *:6379
        $ docker top $ID -axZ
        LABEL           PID     TTY     STAT    TIME    COMMAND
        docker-default  9243    ?       Ssl     0:00    redis-server    *:6379

### Dealing with Images
The following commands provide tools for creating and working with images:

docker build
    Builds an image from a Dockerfile

docker commit
    Creates an image from the specified container. While docker commit can be useful, it is generally preferable to create images using docker build , which is easily repeatable. By default, containers are paused prior to commit, but this can be turned off with the --pause=false argument. Takes -a and -m arguments for setting metadata.
    For example:

        $ ID=$(docker run -d redis touch /new-file)
        $ docker commit -a "Joe Bloggs" -m "Comment" $ID commit:test
        ac479108b0fa9a02a7fb290a22dacd5e20c867ec512d6813ed42e3517711a0cf
        $ docker images commit
        REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
        commit test ac479108b0fa About a minute ago 111 MB
        $ docker run commit:test ls /new-file
        /new-file

docker export
    Exports the contents of the container’s filesystem as a tar archive on STDOUT . The resulting archive can be loaded with docker import . Note that only the filesystem is exported; any metadata such as exported ports, CMD , and ENTRYPOINT settings will be lost. Also note that any volumes are not inlcuded in the export. Contrast with docker save . docker history Outputs information on each of the layers in an image.

docker images
    Provides a list of local images, including information such as repository name, tag name, and size. By default, intermediate images (used in the creation of toplevel images) are not shown. The VIRTUAL SIZE is the total size of the image including all underlying layers. As these layers may be shared with other images, simply adding up the size of all images does not provide an accurate estimate of disk usage. Also, images will appear multiple times if they have more than one tag; different images can be discerned by comparing the ID. Takes several arguments; in particular, note -q , which only returns the image IDs and is useful as input to other commands such as docker rmi .
    For example:

        $ docker images | head -4
        REPOSITORY              TAG     IMAGE ID        CREATED         VIRTUAL SIZE
        identidock_identidock   latest  9fc66b46a2e6    26 hours ago    839.8 MB
        redis                   latest  868be653dea3    6 days ago      110.8 MB
        containersol/pres-base  latest  13919d434c95    2 weeks ago     401.8 MB
        
    To remove all dangling images:

        $ docker rmi $(docker images -q -f dangling=true)
        Deleted: a9979d5ace9af55a562b8436ba66a1538357bc2e0e43765b406f2cf0388fe062
    
docker import
    Creates an image from an archive file containing a filesystem, such as that created by docker export . The archive may be identified by a file path or URL or streamed through STDIN (by using the - flag). Returns the ID of the newly created image. The image can be tagged by supplying a repository and tag name. Note that an image built from import will only consist of a single layer and will lose Docker configuration settings such as exposed ports and CMD values. Contrast with docker load .
    Example of “flattening” an image by exporting and importing:
    
        $ docker export 35d171091d78 | docker import - flatten:test
        5a9bc529af25e2cf6411c6d87442e0805c066b96e561fbd1935122f988086009
        $ docker history flatten:test
        IMAGE CREATED CREATED BY SIZE COMMENT
        981804b0c2b2 59 seconds ago 317.7 MB Imported from -

docker load
    Loads a repository from a tar archive passed via STDIN . The repository may contain several images and tags. Unlike docker import , the images will include history and metadata. Suitable archive files are created by docker save , making save and load a viable alternative to registries for distributing images and producing backups. See docker save for an example.

docker rmi
    Deletes the given image or images. Images are specified by ID or repository and tag name. If a repository name is supplied but no tag name, the tag is assumed to be latest . To delete images that exist in multiple repositories, specify that image by ID and use the -f argument. You will need to run this once per repository.

docker save
    Saves the named images or repositories to a tar archive, which is streamed to STDOUT (use -o to write to a file). Images can be specified by ID or as repository:tag . If only a repository name is given, all images in that repository will be saved to the archive, not just the latest tag. Can be used in conjunction with docker load to distribute or back up images.
    For example:

        $ docker save -o /tmp/redis.tar redis:latest
        $ docker rmi redis:latest
        Untagged: redis:latest
        Deleted: 868be653dea3ff6082b043c0f34b95bb180cc82ab14a18d9d6b8e27b7929762c
        ...

        $ docker load -i /tmp/redis.tar
        $ docker images redis
        REPOSITORY  TAG     IMAGE ID        CREATED         VIRTUAL SIZE
        redis       latest  0f3059144681    3 months ago    111 MB

docker tag
    Associates a repository and tag name with an image. The image can identified by ID or repository and tag (the latest tag is assumed if none is given). If no tag is given for the new name, latest is assumed.
    For example:

        $ docker tag faa2b75ce09a newname
        $ docker tag newname:latest bryanagamk/newname
        $ docker tag newname:latest bryanagamk/newname:newtag
        $ docker tag newname:latest myregistry.com:5000/newname:newtag

1. Adds the image with ID faa2b75ce09a to the repository newname , using the tag latest as none was specified.
1. Adds the newname:latest image to the bryanagamk/newname repository, again using the tag latest. This label is in a format suitable for pushing to the Docker Hub, assuming the user is bryanagamk .
1. As above except using the tag newtag instead of latest .
1. Adds the newname:latest image to the repository myregistry.com/newname with the tag newtag . This label is in a format suitable for pushing to a registry at http://myregistry.com:5000.(((range="endofrange”, startref="ix_04_docker_fundamentals-asciidoc23”)))(((range="endofrange”, startref="ix_04_docker_fundamentals-asciidoc22”)))

### Using the Registry
The following commands relate to using registries, including the Docker Hub. Be aware the Docker saves credentials to the file .dockercfg in your home directory:

docker login
    Register with, or log in to, the given registry server. If no server is specified, it is assumed to be the Docker Hub. The process will interactively ask for details if required, or they can be supplied as arguments

docker logout
    Logs out from a Docker registry. If no server is specified, it is assumed to be the Docker Hub.

docker pull
    Downloads the given image from a registry. The registry is determined by the image name and defaults to the Docker Hub. If no tag name is given, the image tagged latest will be downloaded (if available). Use the -a argument to download all images from a repository.

docker push
    Pushes an image or repository to the registry. If no tag is given, this will push all images in the repository to the registry, not just the one marked latest .

docker search
    Prints a list of public repositories on the Docker Hub matching the search term. Limits results to 25 repositories. You can also filter by stars and automated builds. In general, it’s easiest to use the website.
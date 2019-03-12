# First Step

## Running Your First Image
To test Docker is installed correctly, try running:
`   $ docker run debian echo "Bye World"`

This may take a little while, depending on your Internet connection    
We’ve called the docker run command, which is responsible for launching containers. The argument debian 
is the name of the image we want to use—in this case, a stripped-down version of the Debian Linux distribution

The first line of the output tells us we don’t have a local copy of the Debian image. Docker then checks 
online at the Docker Hub and downloads the newest version of the Debian image

Once the image has been downloaded, Docker turns the image into a running container and executes the command 
we specified— echo "Bye World" inside it
If you run the same command again, it will immediately launch the container without downloading. 

We can ask Docker to give us a shell inside a container with the following command:

```shell
    $ docker run -i -t debian /bin/bash
    root@622ac5689680:/# echo "Hello dari dalam Container"
    Hello dari dalam Container
    root@622ac5689680:/# exit
    exit
```

## The Basic Commands
Let’s try to understand Docker a bit more by launching a container and seeing what effect various
commands and actions have. 

First, let’s launch a new container; but this time, we’ll give it a new hostname with the -h flag:
`   $ docker run -h CONTAINER -i -t debian /bin/bash`
`   root@CONTAINER:/#`

Open a new terminal (leave the container session running), and try running docker ps from the host. 
You will see something like this:
```shell
    CONTAINER ID    IMAGE   COMMAND     ...     NAMES
    00723499fdbf    debian  "/bin/bash" ...     stupefied_turing
```

This tells us a few details about all the currently running containers.
We can get more information on a given container by runningdocker inspect with the name or ID of the 
container:
```shell
    $ docker inspect stupefied_turing
        [
        {
            "Id": "00723499fdbfe55c14565dc53d61452519deac72e18a8a6fd7b371ccb75f1d91",
            "Created": "2015-09-14T09:47:20.2064793Z",
            "Path": "/bin/bash",
            "Args": [],
            "State": {
            "Running": true,
        ...
```    

There is a lot of valuable output here, but it’s not exactly easy to parse. We can use grep or the 
--format argument (which takes a Go template 4 ) to filter for the information we’re interested in. 
For example:
```shell
     $ docker inspect stupefied_turing | grep IPAddress
            "IPAddress": "172.17.0.4",
            "SecondaryIPAddresses": null,

        $ docker inspect --format {{.NetworkSettings.IPAddress}} stupefied_turing
            172.17.0.4
```    

docker diff :
```shell
    $ docker diff stupefied_turing
        C /.wh..wh.plnk
        A /.wh..wh.plnk/101.715484
        D /bin
        A /basket
        A /basket/bash
        A /basket/cat
        A /basket/chacl
        A /basket/chgrp
        ...    
```

Docker logs . If you run this command with the name of your container, you will get a list of everything that’s happened inside the container:
```shell
    $ docker logs stupefied_turing
        root@CONTRAINER:/# mv /bin /basket
        root@CONTRAINER:/# ls
        bash: ls: command not found   
```

We’re finished with our broken container now, so let’s get rid of it. First, exit from the shell:
```shell
    root@CONTRAINER:/# exit
    exit
    $    
```
    
This will also stop the container, since the shell was the only running process. If you run docker ps , you should see there are no running containers. (officially called exited containers) To get rid of the container, use the docker rm command:
```shell
    $ docker rm stupefied_turing
    stupefied_turing    
```

We’re going to create a Dockerized cowsay application
```shell
    $ docker run -it --name cowsay --hostname cowsay debian bash
    root@cowsay:/# apt-get update
        ...
    Reading package lists... Done
    root@cowsay:/# apt-get install -y cowsay fortune
    ...
    root@cowsay:/#    
```
    
Give it a whirl!

```shell
    root@cowsay:/# /usr/games/fortune | /usr/games/cowsay
        _____________________________________
        / Writing is easy; all you do is sit \
        | staring at the blank sheet of paper |
        | until drops of blood form on your   |
        | forehead.                           |
        |                                     |
        \ -- Gene Fowler                     /
        -------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\        )\/\
                ||----w  |
                ||      ||    
```
   
Let’s keep this container. 6 To turn it into an image, we can just use the docker commit command. To do this, we need to give the command the name of the container (“cowsay”) a name for the image (“cowsayimage”) and the name of the repository to store it in (“test”):
```shell
    root@cowsay:/# exit
        exit
        $ docker commit cowsay test/cowsayimage
        d1795abbc71e14db39d24628ab335c58b0b45458060d1973af7acf113a0ce61d   
```

## Building Images from Dockerfiles
A Dockerfile is simply a text file that contains a set of steps that can be used to create
a Docker image. Start by creating a new folder and file for this example:
 ```shell
    $ mkdir cowsay
    $ cd cowsay
    $ touch Dockerfile    
```

And insert the following contents into Dockerfile:
```dockerfile
    FROM debian:wheezy
    RUN apt-get update && apt-get install -y cowsay fortune    
```

We can now build the image by running the docker build command inside the same directory:
```shell
    $ ls
    Dockerfile
    $ docker build -t test/cowsay-dockerfile .
    Sending build context to Docker daemon 2.048 kB
    Step 0 : FROM debian:wheezy
        ---> f6fab3b798be
    Step 1 : RUN apt-get update && apt-get install -y cowsay fortune
        ---> Running in 29c7bd4b0adc
    ...
    Setting up cowsay (3.03+dfsg1-4) ...
        ---> dd66dc5a99bd
    Removing intermediate container 29c7bd4b0adc
    Successfully built dd66dc5a99bd        
```

Then we can run the image in the same way as before:
`   $ docker run test/cowsay-dockerfile /usr/games/cowsay "Moo"`

But we can actually make things a little bit easier for the user by taking advantage of 
the ENTRYPOINT Dockerfile instruction. The ENTRYPOINT instruction lets us specify an
executable that is used to handle any arguments passed to docker run .
    
Add the following line to the bottom of the Dockerfile:
`   ENTRYPOINT ["/usr/games/cowsay"]`

    We can now rebuild and run the image without needing to specify the cowsay command:
```shell
    $ docker build -t test/cowsay-dockerfile .
    ...
    $ docker run test/cowsay-dockerfile "Moo"
    ...
```
Much easier! But now we’ve lost the ability to use the fortune command inside the
container as input to cowsay. We can fix this by providing our own script for the
ENTRYPOINT , which is a common pattern when creating Dockerfiles. 
    
Create a file entrypoint.sh with the following contents and save it in the same directory as the
Dockerfile:
```bash
    #!/bin/bash
    if [ $# -eq 0 ]; then
        /usr/games/fortune | /usr/games/cowsay
    else
        /usr/games/cowsay "$@"
    fi 
```
    
Set the file to be executable with chmod +x entrypoint.sh

We next need to modify the Dockerfile to add the script into the image and call it with the ENTRYPOINT
instruction. Edit the Dockerfile so that it looks like:
``` dockerfile   
    FROM debian
    RUN apt-get update && apt-get install -y cowsay fortune
    COPY entrypoint.sh /
    ENTRYPOINT ["/entrypoint.sh"]   
```
    
    Try building a new image and running containers with and without arguments:
``` shell
    $ docker build -t test/cowsay-dockerfile .
        ...snip...
    $ docker run test/cowsay-dockerfile
         ____________________________________
        / The last thing one knows in        \
        | constructing a work is what to put  |
        | first.                              |
        |                                     |
        \       -- Blaise Pascal             /
          ------------------------------------
            \   ^__^
             \  (oo)\_______
                (__)\        )\/\
                    ||----w  |
                    ||      ||    
```

## Working with Registries
Now that we’ve created something amazing, how can we share it with others? When
we first ran the Debian image at the start of the chapter, it was downloaded from the
official Docker registry—the Docker Hub

### Registries, Repositories, Images, and Tags
There is a hierarchical system for storing images. The following terminology is used:

* **Registry**
    - A service responsible for hosting and distributing images. The default registry is the Docker Hub.
* **Repository**
    - A collection of related images (usually providing different versions of the same application or service).
* **Tag**
    - An alphanumeric identifier attached to images within a repository (e.g., 14.04 or stable ).

So the command docker pull amouat/revealjs:latest will download the image
tagged latest within the amouat/revealjs repository from the Docker Hub registry.

In order to upload our cowsay image, you will need to sign up for an account with the
Docker Hub (either online or using the docker login command). After you have
done this, all we need to do is tag the image into an appropriately named repository
and use the docker push command to upload it to the Docker Hub

But first, let’s add a MAINTAINER instruction to the Dockerfile, which simply sets the author contact
information for the image:

```dockerfile
    FROM debian
    MAINTAINER John Smith <john@smith.com>
    RUN apt-get update && apt-get install -y cowsay fortune
    COPY entrypoint.sh /
    ENTRYPOINT ["/entrypoint.sh"]
```

Now let’s rebuild the image and upload it to the Docker Hub. This time, you will need
to use a repository name that starts with your username on the Docker Hub (in my
case, bryanagamk ), followed by / and whatever name you want to give the image. For
example:
```shell
    $ docker build -t bryanagamk/cowsay .
    ...
    $ docker push bryanagamk/cowsay
```


## Using the Redis Official Image

In this case, we’ll have a look at the offical image for Redis, a popular key-value
store.

Start by getting the image:
```shell
    $ docker pull redis
    Using default tag: latest
    latest: Pulling from library/redis
    d990a769a35e: Pull complete
    ...
```

Start up the Redis container, but this time use the -d argument:
```shell
    $ docker run --name myredis -d redis
    585b3d36e7cec8d06f768f6eb199a29feb8b2e5622884452633772169695b94a
```

Ok, so how do we use it? Obviously we need to connect to the database in some way.
We don’t have an application, so we’ll just use the redis-cli tool. We could just
install the redis-cli on the host, but it’s easier and more informative to launch a new
container to run redis-cli in and link the two:

```shell
    $ docker run --rm -it --link myredis:redis redis /bin/bash
    root@ca38735c5747:/data# redis-cli -h redis -p 6379
    redis:6379> ping
    PONG
    redis:6379> set "abc" 123
    OK
    redis:6379> get "abc"
    "123"
    redis:6379> exit
    root@ca38735c5747:/data# exit
    exit
```

we’ve just linked two containers and added some data to Redis in a few
seconds. The linking magic happened with the --link myredis:redis argument to docker
run . This told Docker that we wanted to connect the new container to the existing
“myredis” container, and that we want to refer to it by the name “redis” inside our
new container.

To achieve this, Docker set up an entry for “redis” in /etc/hosts inside the container, 
pointing to the IP address of the “myredis”. This allowed us to use the hostname “redis” 
in the redis-cli rather than needing to somehow pass in, or discover, the IP address of 
the Redis container.

Docker provides how do we persist and back up our data, through the concept of volumes. Volumes are files or
directories that are directly mounted on the host and not part of the normal union
file system. This means they can be shared with other containers and all changes will
be made directly to the host filesystem

Volumes are files or directories that are directly mounted on the host and not part of the normal union file system. 
This means *they can be shared with other containers and all changes will be made directly to the host filesystem*.

There are two ways of declaring a directory as a volume, either using the VOLUME instruction 
inside a Dockerfile or specifying the -v flag to docker run . Both the following Dockerfile 
instruction and docker run command have the effect of creating a volume as /data inside a container: 

`   VOLUME /data`
and
`   $ docker run -v /data test/webserver`

By default, the directory or file will be mounted on the host inside your Docker
installation directory (normally /var/lib/docker/).

So, how do we use this to do backups with the Redis container? The following shows
one way, assuming the myredis container is still running:
```shell
    $ docker run --rm -it --link myredis:redis redis /bin/bash
    root@09a1c4abf81f:/data# redis-cli -h redis -p 6379
    redis:6379> set "persistence" "test"
    OK
    redis:6379> save
    OK
    redis:6379> exit
    root@09a1c4abf81f:/data# exit
    exit
    $ docker run --rm --volumes-from myredis -v $(pwd)/backup:/backup \
    debian cp /data/dump.rdb /backup/
    $ ls backup
    dump.rdb
```

Note that we have used the -v argument to mount a known directory on the host and
--volumes-from to connect the new container to the Redis database folder.

Once you’ve finished with the myredis container, you can stop and delete it:
```shell    
    $ docker stop myredis
    myredis
    $ docker rm -v myredis
    myredis
```

And you can remove all leftover containers with:
```
    $ docker rm $(docker ps -aq)
    45e404caa093
    e4b31d0550cd
    7a24491027fc
    ...
```






























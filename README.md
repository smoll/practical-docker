# practical-docker

The original presentation can be found [here](https://docs.google.com/presentation/d/1L9d2QSmskAlNP9XUWznLpRJ7a5PqoJpFKNdr5w1f2oU/edit#slide=id.p).

## ToC

* [Prerequisites](./PREREQUISITES.md)

### Getting started

* [Running an image from Docker Hub](#running-an-image-from-docker-hub)
* [Clean up a running container](#clean-up-a-running-container)
* [Explicitly mapping ports](#explicitly-mapping-ports)

### Digging deeper

* [Building a Dockerfile locally](#building-a-dockerfile-locally)
* [Poking around a running container](#poking-around-in-a-running-container)
* [Setting env vars inside the container](#setting-env-vars-inside-the-container)

### Using other tools

* [Docker Compose](#docker-compose)
* [docker-gc](#docker-gc)

## Assumptions

0. You are on a Mac
0. When a code snippet is given, lines prefixed with `$` means they are entered on the command line, and the output is usually listed below it. Lines prefixed with `#` are comments about the code immediately below it.
0. You have a working Docker daemon running on a Docker Machine VM (`docker ps` and `docker login` both succeed) -- if not, follow the directions in prerequisites.

## hello-world

Let's look at https://github.com/tutumcloud/hello-world

First we cloned the repo:

```bash
$ cd ~/workspace
$ git clone https://github.com/tutumcloud/hello-world
$ cd hello-world
```

From tutum's README, we have 2 ways of running the hello-world container -- running it directly from Docker Hub, or building it locally then running the image we build.

### Running an image from Docker Hub

```bash
$ docker run -d -p 80 tutum/hello-world
```
```bash
Unable to find image 'tutum/hello-world:latest' locally
latest: Pulling from tutum/hello-world
d6ead20d5571: Pull complete
f4505a442249: Pull complete
40e07714766d: Pull complete
28dcda553f43: Pull complete
73572a17aa2c: Pull complete
93c45db0d698: Pull complete
4b95f40f2f4d: Pull complete
Digest: sha256:0d57def8055178aafb4c7669cbc25ec17f0acdab97cc587f30150802da8f8d85
Status: Downloaded newer image for tutum/hello-world:latest
946dfbcc76930422b6534e6d7cf2437b43918f5604286616c3ce8ed2e8444634
```

There are a few things happening here:

* Because we are on a clean boot2docker VM, there are no docker images on disk.

* We asked for an image called `tutum/hello-world` and Docker inferred that it could simply pull it from Docker Hub, which it did.

* We didn't explicitly add a tag to the desired image (like `tutum/hello-world:1.0`) so Docker's default behavior is to pull the tag labeled `latest`.

* We passed in a port flag `-p 80` which says to publish the internal container port of `80`

Now the container should be up and running:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
946dfbcc7693        tutum/hello-world   "/bin/sh -c 'php-fpm "   3 minutes ago       Up 3 minutes        0.0.0.0:32768->80/tcp   modest_jones
```

Let's see if we can use curl to hit the container. First, we find the IP of the Docker VM (since we are on a Mac, it isn't just `localhost`):

```bash
$ docker-machine ip default
192.168.99.100
```

Now let's try hitting it on port 80:

```bash
$ curl http://192.168.99.100:80
curl: (7) Failed to connect to 192.168.99.100 port 80: Connection refused
```

It doesn't work because by default `-p` will automatically assign a random, free "ephemeral port" within a certain range. If you scrutinize the `docker ps` output above, it's actually listening on port `32768`. Let's take this opportunity to clean up the running container for now.

### Clean up a running container

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                   NAMES
946dfbcc7693        tutum/hello-world   "/bin/sh -c 'php-fpm "   8 minutes ago       Up 8 minutes        0.0.0.0:32768->80/tcp   modest_jones
```

After we know the container ID,

```bash
$ docker kill 946dfbcc7693
946dfbcc7693

$ docker rm 946dfbcc7693
946dfbcc7693

# To do it all in one command we could have also done
$ docker rm -f 946dfbcc7693
```

### Explicitly mapping ports

Let's explicitly map the port by using a slightly modified command:

```bash
$ docker run -d -p 80:80 tutum/hello-world
9387a4a13f93043ffc5e7c959816d8702e64f1e449cb2b0e89d0d64f69a0b774

# Let's confirm it's listening on port 80 now
$ curl http://192.168.99.100:80 
<html>
<head>
	<title>Hello world!</title>
# ...
```

Let's continue our learning by building the image locally.

### Building a Dockerfile locally

Again, referring back to the tutum [README](https://github.com/tutumcloud/hello-world) we can build and run locally by simply doing:

```bash
# Build and tag our image
$ docker build -t smoll/hello-world .
```
```bash
Sending build context to Docker daemon 184.8 kB
Step 1 : FROM alpine
latest: Pulling from library/alpine
Digest: sha256:78a756d480bcbc35db6dcc05b08228a39b32c2b2c7e02336a2dcaa196547a41d
Status: Downloaded newer image for alpine:latest
 ---> 74e49af2062e
Step 2 : MAINTAINER support@tutum.co
 ---> Using cache
 ---> 4aebfb22301c
Step 3 : RUN apk --update add nginx php-fpm &&     mkdir -p /var/log/nginx &&     touch /var/log/nginx/access.log &&     mkdir -p /tmp/nginx &&     echo "clear_env = no" >> /etc/php/php-fpm.conf
 ---> Using cache
 ---> abf01db97deb
Step 4 : ADD www /www
 ---> b12c8cba2d70
Removing intermediate container 2b4f24696e89
Step 5 : ADD nginx.conf /etc/nginx/
 ---> c03e7ef207e3
Removing intermediate container d4f1f4502025
Step 6 : EXPOSE 80
 ---> Running in 8e236e9847ac
 ---> 5aa618efafc1
Removing intermediate container 8e236e9847ac
Step 7 : CMD php-fpm -d variables_order="EGPCS" && (tail -F /var/log/nginx/access.log &) && exec nginx -g "daemon off;"
 ---> Running in 6b1c0dda867e
 ---> 9f11a55bcea0
Removing intermediate container 6b1c0dda867e
Successfully built 9f11a55bcea0
```

Because I've built it before, and didn't change any of the lines in my Dockerfile, or modify any of the files the `RUN` instructions refer to, Docker intelligently uses the build cache and builds my image almost instantly. If I were to invalidate this cache, you would see a lot more verbose output indicating that packages are being fetched.

Let's run the image now

```bash
$ docker run -d -p 80:80 smoll/hello-world
```

It should behave identically to the one we pulled from Docker Hub.

We could even push it to Docker Hub under our own name:

```bash
$ docker push smoll/hello-world
```
```bash
The push refers to a repository [docker.io/smoll/hello-world] (len: 1)
9f11a55bcea0: Pushed
5aa618efafc1: Pushed
c03e7ef207e3: Pushed
b12c8cba2d70: Pushing [==================================================>]  16.9 kB
```

Docker automatically creates a Docker Hub repo for it too, if it doesn't yet exist (as long as we did `docker login` as mentioned above.)

### Poking around in a running container

The way I usually do this is:

```bash
# Get the container id of the running container
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
0cb9dd2f6f01        smoll/hello-world   "/bin/sh -c 'php-fpm "   6 minutes ago       Up 6 minutes        0.0.0.0:80->80/tcp   serene_ardinghelli
```
After we know the container ID,

```bash
# Start an interactive shell
$ docker exec -it 0cb9dd2f6f01 bash
exec: "bash": executable file not found in $PATH
```

where `-i` says to start an interactive shell and `-t` means allocate a pseudo-TTY. You can combine `-i -t` into just `-it`.

Unfortunately, since the base image for this container was Alpine Linux, a minimal distro, it doesn't even have `bash`. We can use `sh`, however:

```bash
$ docker exec -it 0cb9dd2f6f01 /bin/sh

(inside the container)$ ls
bin      dev      etc      home     lib      linuxrc  media    mnt      proc     root     run      sbin     sys      tmp      usr      var      www

(inside the container)$ uname -a
Linux 0cb9dd2f6f01 4.1.13-boot2docker #1 SMP Fri Nov 20 19:05:50 UTC 2015 x86_64 Linux
```

The `uname` command shows us that the container knows the OS distro of the host Docker daemon (boot2docker) and not the OS of the Docker image (Alpine Linux).

### Setting env vars inside the container

Environment variables are set by passing them in the `docker run` command. For example:

```bash
$ docker run -d -p 80:80 -e "HELLO=true" -e "BYE=false" smoll/hello-world
3c0f3dac333e286d5665cf29386528433d73e51237f3a365558dcfd4e96a3111
```

We can check the env vars are set by reading through the output of the inspect command:

```bash
# Note I only need to write the first few chars of the container id
$ docker inspect 3c0f
# ...
        "Env": [
            "HELLO=true",
            "BYE=false",
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ],
```

Or alternatively, exec in and echo them:

```bash
# Can use short or long form of container id
$ docker exec -it 3c0f3dac333e sh

(inside the container)$ echo $HELLO
true

(inside the container)$ echo $BYE
false
```

## Docker Compose

Having to construct the Docker run commands by hand can get cumbersome, especially when there are a lot of env vars that need to be set. Instead we can use Docker Compose to define them in a simple YML format.

TODO: fill this out

## docker-gc

TODO!

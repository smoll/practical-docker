# practical-docker

## Prerequisites

Install Docker Toolbox, which comes with Docker Machine:
* [Step-by-step walkthrough](https://docs.docker.com/mac/step_one/)
* [Download link](https://www.docker.com/docker-toolbox)

Ensure that docker-machine (boot2docker/Vagrant/VirtualBox VM) is created and started:

```bash
$ docker-machine ls
NAME      ACTIVE   DRIVER       STATE     URL   SWARM   DOCKER    ERRORS
default   -        virtualbox   Stopped                 Unknown
```

If it shows stopped (like above),

```bash
$ docker-machine start default
(default) Starting VM...
Machine "default" was started.
Started machines may have new IP addresses. You may need to re-run the `docker-machine env` command.
```

NOTE: your docker-machine might be called "dev" or something else instead of "default"

For convenience, I have added the following to `~/.bash_profile` so it always exports the docker-machine envs to my current shell as soon as I open a new iTerm window:

```bash
echo "exporting Docker Machine env to shell..."
eval "$(docker-machine env default </dev/null)"
```

Now, ensure that you can access the docker daemon:

```bash
$ docker ps
```

The best check for connectivity between the daemon and the Internet (sometimes it breaks when going from the office to home, etc.) is

```bash
$ docker login
```

If this hangs, restart the docker machine.

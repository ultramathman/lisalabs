# Docker Basics
>**Author:** Steve Anthony (sma310@lehigh.edu)
>
>USENIX LISA Lab 2017

----------
**Table of Contents**

- [Lab Objective](#lab-objective)
- [Suggested Knowledge Prerequisites](#suggested-knowledge-prerequisites)
- [Suggested Physical Prerequisites](#suggested-physical-prerequisites)
- [Let's Begin!](#lets-begin)
  - [Configure Networking for VirtualBox](#configure-networking-for-virtualbox)
  - [Installing Docker](#installing-docker)
  - [Getting a Container](#getting-a-container)
  - [Start the Container Interactively](#start-the-container-interactively)
  - [View Container Information](#view-container-information)
  - [Create Persistent Storage](#create-persistent-storage)
  - [Run the Container with Storage and Network](#run-the-container-with-storage-and-network)
  - [Enable Access from the VM Host](#enable-access-from-the-vm-host)
- [What's Next?](#whats-next)
## Lab Objective
In this lab you will learn how to install Docker, find and download a container, connect and disconnect from that container, and attach networking and storage.

----------
## Suggested Knowledge Prerequisites
 
 - Familiarity with Linux and working from a command line environment.
 - Familiarity with [VirtualBox](https://www.virtualbox.org/wiki/Downloads) helpful, but not required.

 
 ----------
## Suggested Physical Prerequisites
1. A computer able to run a virtual machine with 512MB of RAM and 1 core. You will need to install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) version 4.3+ if you don't already have it.

----------
## Let's Begin!

### Configure Networking for VirtualBox
VirtualBox offers several networking options. In order to make the Influx labs easier we will use the same network type for both sections. This will require us to make some initial network configuration changes to use the "NAT Network" network type.

> **Note:** VirtualBox offers two similarly named network types, "NAT" and "NAT Network".  The default "NAT" network type allows VMs to access the Internet, but not address other VMs. The "NAT Network" network option we're using will allow our VMs to connect both to the Internet and be addressable by other VMs we'll run in the "HA with InfluxDB" followup lab.

We configure the `NAT Network` as follows:

1. In the VirtualBox main window, click on `File`, then `Preferences`. 
2.  Click on the `Network` section.
3. Create a `NAT Network` if one does not exist. 

Now change the `docker` VM to use the new network.

1. Click on the `docker` VM and then click `Settings`.
2. Click on the `Network` section and change the adapter to be `Attached to` the NATNetwork you created.

### Installing Docker
Launch the "docker" VM and log in. The account has sudo privileges.
 
> **Credentials:**
>
> Username: student
>
> Password: brainfood!

Install the prerequisite software:
```bash
sudo apt-get install \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common
```
Next, add the docker key and repository.
```bash
$ curl -sL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
$ sudo -s
cat > /etc/apt/sources.list.d/docker.list
deb [arch=amd64] https://download.docker.com/linux/debian stretch stable
```
Now, update apt, install the Docker Community Edition, and add the `student` user to the `docker` group so we can run our container as a unprivileged user. 
```bash
$ sudo apt-get update
$ sudo apt-get install docker-ce
$ sudo adduser student docker
```
>**Note:** You will need to log out and back in so the group membership changes will take effect.

### Getting a Container
The [DockerHub](https://hub.docker.com/explore/) provides a listing of official repositories for various projects and distributions, along with instruction describing their use. In this lab, we will be using the [Debian](https://hub.docker.com/_/debian/) container, specifically the `stretch` release, which matches the OS installed in the VM.

Docker will automatically download the image for the container when you attempt to run it for the first time. If you'd like to download the image without starting a container, you can do so as follows.
```bash
$ docker image pull debian:stretch
```

### Start the Container Interactively
As containers are often used to run a single application or piece of an application stack, many are not run interactively. For the purposes of exploration, we'll run our Debian container with bash to attach our shell to the container.
```bash
$ docker run -it debian:stretch bash
root@dd69a20e1d6f:/#
```
We now have a root shell in the container environment with ID `dd69a20e1d6f`. Let's install Apache and OpenSSH.
```bash
apt update
apt install apache2 openssh-server
```

Now we can detach our terminal from the container using `Ctrl+p, Ctrl+q`. This will return us to our host environment and leave the container running.
### View Container Information
Containers can be manipulated using the subcommands of `docker container`. For example, to see the containers running, we can use the following.
```bash
$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
76d4eff9c4a8        debian:stretch      "bash"              30 minutes ago      Up 30 minutes                           eager_lamarr
```
>**Note:** Unless we explicitly name our container using the `--name` flag, Docker will choose a random name for us. In this case, `eager_lamarr`.

We can see additional information about our container using the `inspect` subcommand.
```bash
$ docker container inspect eager_lamarr
[
    {
        "Id": "76d4eff9c4a81e8198b8cf5d04880048596569ddd6d306e750fb2a1b0175ba87",
        "Created": "2017-10-27T15:19:33.399097647Z",
        "Path": "bash",
        "Args": [],
        "State": {
...snip...
```
Notice the container is using the `bridge` network. Note the first 13 characters of the `NetworkID`, and that `Mounts` section and the `Ports` subsection of `NetworkSettings` in the output are empty.
>**Note:** To view more information about the network setup use the subcommands of `docker network`.

At this point we have a Debian container running web and SSH services. However, anything we create in our webroot will be destroyed if we remove the container. To solve this, we will mount persistent storage for /var/www and serve a page there.

### Create Persistent Storage
Storage is managed using the subcommands of `docker storage`. Use this command to create a `lisa17` volume for our data. Then use the `inspect` subcommand to get more information about our new volume.
```bash
$ docker volume create lisa17
lisa17
$ docker volume inspect lisa17
[
    {
        "CreatedAt": "2017-10-27T11:28:05-07:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/lisa17/_data",
        "Name": "lisa17",
        "Options": {},
        "Scope": "local"
    }
]
```
>**Note:** Notice the `Mountpoint` above. This is where data will be stored on the *host* not where it will be located inside the container. We will choose a mountpoint when we attach it to the new container.

### Run the Container with Storage and Network
We now need to attach the persistent storage volume we created to our container. Unfortunately, we can only do this when we initially provision the container. Remember, containers themselves are meant to be ephemeral, so lets stop our container.
```bash
$ docker container stop eager_lamarr
eager_lamarr
$ docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
While we're creating our new container, we also will forward ports `80` to `8080` and `22` to `2222` on the container host. In the next section, we'll from Virtual Box to the VM host.
```bash
docker run -it -p 8080:80 -p 2222:22 --mount source=lisa17,destination=/var/www debian:stretch bash
root@46ce91515e8f:/#
```

>**Note:** Since this is a new container, we'll need to update apt and install `apache2` and `openssh-server` again. Additionally install an editor, eg. `emacs25-nox` or `vim`.

Set a root password in the container with `passwd` and edit `/etc/ssh/sshd_config`, changing `PermitRootLogin` to `yes` so we can SSH into the container. 

Feel free to replace `/var/www/html/index.html` with your own content as well.

### Enable Access from the VM Host
To enable access to the our webpage from the VM host (ie. your computer) we need to forward a port to VirtualBox which will map it onto the VM port 8080. This is configured as follows:

1. In the VirtualBox main window, click on `File`, then `Preferences`. 
2. Click on the `Network` section.
3. Edit your `NAT Network`.
4. Click `Port Forwarding`.
5. Add a rule to forward host port 8080 (IP address blank) to guest port 8080 (enter the IP of the `docker` host).

Likewise add a rule to forward port `2222` to port `2222` for SSH.

At this point able to visit `http://localhost:8080` and see the webpage running inside our Docker container. Using an SSH client on the VM host on port `2222` we can also log into our container to make further changes. Congratulations! 

## What's Next?
We have a few suggestions for what to do next.  You can try to reimplement “LISA USENIX Lab 2017 - HA InfluxDB” using Docker containers for the InfluxDB, influx-relay, and HAproxy instances, apply this same idea to one of the other labs, or start to investigate orchestration and management of Docker containers.

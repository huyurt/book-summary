# docker-pg

## Install Docker Engine on Ubuntu

> **- apt-get -**
>
> *apt-get is the command-line tool for handling packages.*

Older versions of Docker were called `docker`, `docker.io`, or `docker-engine`. If these are installed, uninstall them:

```bash
sudo apt-get remove docker docker-engine docker.io conrainerd runc
```



> **- curl -**
>
> *Curl is a command-line tool for transferring data specified with URL syntax.*
>
> ```bash
> sudo apt install curl
> ```

Install using the convenience script:

```bash
sudo curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```



Show the Docker version information:

```bash
sudo docker version
```



Run the Docker [demo tutorial](https://hub.docker.com/r/docker/whalesay):

```bash
sudo docker run docker/whalesay cowsay Hello-World
```



## Basic Docker Commands

### run

The `docker run` command first `creates` a writeable container layer over the specified image, and then `starts` it using the specified command. That is, `docker run` is equivalent to the API `/containers/create` then `/containers/(id)/start`.

```bash
sudo docker run image_name

# run container in background
sudo docker run -d image_name
```



The `-it` instructs Docker to allocate a pseudo-TTY connected to the container’s stdin; creating an interactive `bash` shell in the container.

```bash
sudo docker run -it image_name
```



Keep STDIN open even if not attached.

```bash
sudo docker run -i image_name
```



Publish a container's port(s) to the host.

```bash
# This binds port 5000 of the container to port 80 on host machine
sudo docker run -p 80:5000 image_name
```



Bind mount a volume.

```bash
# The -v flag mounts the current working directory into the container.
sudo docker run -v /opt/datadir:/path image_name
```



### ps

List containers.

```bash
# only shows running containers by default
sudo docker ps

# see all containers
sudo docker ps -a
```



### stop

Stop one or more running containers.

```bash
sudo docker stop container_name
```



### rm

Remove one or more containers.

```bash
sudo docker rm container_name
```



### images

The default `docker images` will show all top level images, their repository and tags, and their size.

```bash
sudo docker images
```



### rmi

Remove one or more images.

```bash
sudo docker rmi image_name
```



### pull

Pull an image or a repository from a registry.

```bash
sudo pull nginx
```



### exec

Run a command in a running container.

```bash
sudo exec container_command
```



### attach

Attach local standard input, output, and error streams to a running container.

```bash
sudo docker attach container_id
```



## inspect

Return low-level information on Docker objects.

```bash
sudo docker inspect container_name
```



### logs

Fetch the logs of a container.

```bash
sudo docker logs container_name
```



# Appendix

## Assign a Static IP to the Ubuntu VPS instance in VirtualBox

> ```bash
> sudo apt install net-tools
> sudo apt install vim
> ifcongif -a
> sudo vim /etc/network/interfaces
> 
> # press i and add this lines to file
> auto enp0s8
> iface enp0s8 inet static
> address 192.168.56.101
> netmask 255.255.255.0
> 
> # press ESC and type :wq
> sudo reboot
> ```



## Install .NET Core Apps on Linux 

> ```bash
> wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
> sudo dpkg -i packages-microsoft-prod.deb
> sudo apt-get update; \
>   sudo apt-get install -y apt-transport-https && \
>   sudo apt-get update && \
>   sudo apt-get install -y dotnet-sdk-3.1
> ```



## Install SQL Server

> ```bash
> wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
> sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/20.04/mssql-server-2019.list)"
> sudo apt-get update
> sudo apt-get install -y mssql-server
> sudo /opt/mssql/bin/mssql-conf setup
> ```



## Git clone on Ubuntu

> ```bash
> sudo apt install git
> git clone https://github.com/?/?.git
> ```



## Useful Commands

> ```bash
> # login as root
> sudo -s
> 
> # create folder
> mkdir folder_name
> 
> # 
> sudo apt install nautilus-admin
> 
> # open port
> sudo iptables -I INPUT 1 -i eth0 -p tcp --dport 9091 -j ACCEPT
> 
> # check file system disk space usage
> df
> 
> # create backup image
> sudo dd if=/dev/sda conv=sync,noerror bs=64K | gzip -c  > /path/backup_image.img.gz
> ```


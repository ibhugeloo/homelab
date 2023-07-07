# Docker-compose files

This repository contains all my docker-compose files for Docker.

Before using any code, make sure to check the official instructions from the documentation to avoid using outdated code !

## 📤 Links to Docker Hub and GitHub

- [portainer-be](https://hub.docker.com/r/portainer/portainer-ee)
- [bitwarden](https://hub.docker.com/r/vaultwarden/server)
- [jellyfin](https://hub.docker.com/r/jellyfin/jellyfin)
- [nextcloud](https://hub.docker.com/_/nextcloud)
- [transmission](https://hub.docker.com/r/linuxserver/transmission)
- [uptime-kuma](https://github.com/louislam/uptime-kuma)
- [vsftpd](https://hub.docker.com/r/fauria/vsftpd)
- [watchtower](https://hub.docker.com/r/containrrr/watchtower)
- [heimdall](https://hub.docker.com/r/linuxserver/heimdall)

## 🔧 Useful Links

- [Cloudflare Tunnel Tutorial](https://www.youtube.com/watch?v=ey4u7OUAF3c&t=416s)
- [Twingate VPN Tutorial](https://www.youtube.com/watch?v=IYmXPF3XUwo)
- [Setup Docker in LXC](https://du.nkel.dev/blog/2021-03-25_proxmox_docker/)
- [Official Docker Documentation](https://docs.docker.com/engine/install/debian/#install-using-the-repository)

## 🐋 Docker Installation for Debian

To begin the installation of Docker on Debian, follow these steps:

```shell
# Update the machine
sudo apt update && sudo apt full-upgrade -y

# Install sudo
sudo apt install sudo

# Update all packages
sudo apt-get update && sudo apt-get upgrade && sudo apt-get dist-upgrade && sudo apt-get autoremove

# Install Docker
sudo apt-get install ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose

# Test Docker
systemctl status docker
docker run hello-world
```

## 💡 Useful Commands

Here are some useful commands for working with Docker:

- Start/stop Docker service: `systemctl start/stop docker`
- Enable Docker service on startup: `systemctl enable docker`
- Pull Docker image: `docker pull`
- List running containers: `docker ps`
- List all containers: `docker ps -a`
- Start/stop a container: `docker start/stop CONTAINER_ID`
- Remove a container: `docker rm CONTAINER_ID`
- Kill a container: `docker kill CONTAINER_ID`
- List Docker images: `docker images`

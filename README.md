# docker-compose files
All my docker-compose files for docker
Always check the official instructions from the docs, for outdated code, before copy & paste !

# 📤 Liens vers docker hub && github
- portainer-be : https://hub.docker.com/r/portainer/portainer-ee
- bitwarden : https://hub.docker.com/r/vaultwarden/server
- jellyfin : https://hub.docker.com/r/jellyfin/jellyfin
- nextcloud : https://hub.docker.com/_/nextcloud
- transmission : https://hub.docker.com/r/linuxserver/transmission
- uptime-kuma : https://github.com/louislam/uptime-kuma
- vsftpd : https://hub.docker.com/r/fauria/vsftpd
- watchtower : https://hub.docker.com/r/containrrr/watchtower
- heimdall : https://hub.docker.com/r/linuxserver/heimdall/

# 🔧 Liens utiles
- Cloudflare-tunnel : https://www.youtube.com/watch?v=ey4u7OUAF3c&t=416s
- Twingate-vpn : https://www.youtube.com/watch?v=IYmXPF3XUwo
- Setup Docker in LXC : https://du.nkel.dev/blog/2021-03-25_proxmox_docker/
- Official docker documentation : https://docs.docker.com/engine/install/debian/#install-using-the-repository

# 🐋 Installer Docker
Pour débuter l’installation de docker sur Debian, on va commencer par une mise à jour de la machine
- apt update && apt full-upgrade -y

Installer sudo
- apt install sudo

Mettre à jour tout les paquets
- apt-get update && apt-get upgrade && apt-get dist-upgrade && apt-get autoremove

Installer Docker
- sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
- curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
- echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \ $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
- sudo apt-get update
- sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose

Test Docker
- systemctl status docker
- docker run hello-world
 
# 💡 Commandes utiles
- systemctl start/stop docker         
- systemctl enable docker              
- docker pull                           
- docker ps                            
- docker ps -a                        
- docker start/stop CONTAINER ID          
- docker rm CONTAINER ID             
- docker kill CONTAINER ID             
- docker images                        

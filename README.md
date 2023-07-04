# docker-compose files
All my docker-compose files for docker

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

# 🐋 Installer docker
Pour débuter l’installation de docker sur Debian, on va commencer par une mise à jour de la machine
- apt update && apt full-upgrade -y

Puis l’installation des dépendances
- apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

Ensuite, on ajout de la clé GPG officielle de Docker
- curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

Ajout du repository Docker dans les sources
- echo \ "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \ $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

Puis on met à jour la liste des sources
- apt update -y

Ensuite, on télécharge le paquet Docker depuis les sources
- apt install -y docker.io docker-compose -y

Enfin on vérifie l’installation de Docker sur la machine
- docker run hello-world

Créer un dossier docker pour toute les installations
- mkdir docker

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

```bash
docker volume create --driver local --opt type=none --opt device=/home/sanjeev/portainer_data --opt o=bind portainer_data


docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts
```
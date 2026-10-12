Calibre server startpu guide on the plex server

``` bash
docker run -d --name calibre-server \
  --restart=unless-stopped \
  -p 8080:8080 -p 8081:8081 -p 9090:9090 \
  -e PASSWORD="ReadLotsBooks" \
  -v /home/sanjeev/calibre_database:/config \
  linuxserver/calibre:latest

docker update --restart=unless-stopped calibre-server
```

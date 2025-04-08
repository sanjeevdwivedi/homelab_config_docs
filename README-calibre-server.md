Calibre server startpu guide on the plex server

``` bash
docker run -d --name calibre-server -p 8080:8080 -p 8081:8081 -p 9090:9090 -e PASSWORD="ReadLotsBooks" -v /home/sanjeev/calibre_database:/config linuxserv
er/calibre:latest
```
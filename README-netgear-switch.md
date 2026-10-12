```bash


replaced the original command so docker will restart the container automatically
docker build . -t tvcontrol
docker run -d --name tvcontrol --restart=unless-stopped -p 5006:5006 tvcontrol

#docker run -d -p 5006:5006 tvcontrol
```

then go to http://plex-garage.lan:5006/ 

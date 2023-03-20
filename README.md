# Install media stack

There are two media stacks available.

`stack-1` This stack contains Jellyfin, Radarr, Sonarr, and SABnzbd.

`stack-2` This stack contains Jellyfin, Radarr, Sonarr, SABnzbd and VPN.

Any one of them can be deployed using --profile option with docker-compose.

```
docker network create mynetwork

# Install Jellyfin, Radarr, Sonarr, and SABnzbd stack
docker-compose --profile stack-1 up -d

# Or, Install Jellyfin, Radarr, Sonarr, SABnzbd and VPN stack
## By default NordVPN is configured. This can be changed to ExpressVPN, SurfShark, OpenVPN or Wireguard VPN by updating docker-compose.yml file. It uses OpenVPN type for all providers.

VPN_SERVICE_PROVIDER=nordvpn OPENVPN_USER=openvpn-username OPENVPN_PASSWORD=openvpn-password SERVER_REGIONS=Switzerland docker-compose --profile stack-2 up -d

docker-compose -f docker-compose-nginx.yml up -d # OPTIONAL to use Nginx as reverse proxy
```

# Configure SABnzbd

- Open SABnzbd at http://localhost:8080. 
- Configure using wizard, inputting your Usenet provider API key when instructed.

- From backend, Run below commands

```
# docker exec -it sabnzbd bash # Get inside SABnzbd container.

mkdir /downloads/movies /downloads/tvshows
chown 1000:1000 /downloads/movies /downloads/tvshows
```

# Configure Radarr

- Open Radarr at http://localhost:7878
- Settings --> Media Management --> Check mark "Movies deleted from disk are automatically unmonitored in Radarr" under File management section --> Save
- Settings --> Indexers --> Add --> Newznab # Configure based on your Usenet indexer of choice.
- Settings --> Download clients --> SABnzbd --> Add Host and port 8080 --> Username and password if added --> Test --> Save 
**Note: If VPN is enabled, then SABnzbd is reachable on vpn's service name** \ 
**Note: Sometimes test will fail if value is left as 'localhost' It may be necessary to input your IP address.**
- Settings --> General --> Enable advance setting --> Select Authentication and add username and password
# Add a movie

- Movies --> Search for a movie --> Add Root folder (/downloads) --> Quality profile --> Add movie
- Go to SABnzbd (http://localhost:8080) and see if movie is getting downloaded.

# Configure Jellyfin

- Open Jellyfin at http://localhost:8096
- Configure as it asks for first time.
- Add media library folder and choose /downloads/movies. Repeat and choose folder /downloads/tvshows. Name libraries accordingly.

# Apply SSL in Nginx

- Open port 80 and 443.
- Get inside Nginx container and install certbot and certbot-nginx `apk add certbot certbot-nginx`
- Add URL in server block. e.g. `server_name  localhost armdev.navratangupta.in;` in /etc/nginx/conf.d/default.conf
- Run `certbot --nginx` and provide details asked.


# Configure Nginx

- Get inside Nginx container
- `cd /etc/nginx/conf.d`
- Add proxies as per below for all tools.
- OR, copy nginx.conf file to /etc/nginx/conf.d/default.conf and make necessary changes

`docker cp nginx.conf nginx:/etc/nginx/conf.d/default.conf && docker exec -it nginx nginx -s reload`
- Close ports of other tools in firewall/security groups except port 80 and 443.


# Radarr Nginx reverse proxy

- Settings --> General --> URL Base --> Add base (/radarr)
- Add below proxy in nginx configuration

```
location /radarr {
    proxy_pass http://radarr:7878;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
```

- Restart containers.

# Sonarr Nginx reverse proxy

- Settings --> General --> URL Base --> Add base (/sonarr)
- Add below proxy in nginx configuration

```
location /radarr {
    proxy_pass http://sonarr:8989;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
  }
```


- Restart containers.


# Jellyfin Nginx proxy

- Add base URL, Admin Dashboard -> Networking -> Base URL (/jellyfin)
- Add below config in Ngix config

```
 location /jellyfin {
        return 302 $scheme://$host/jellyfin/;
    }

    location /jellyfin/ {

        proxy_pass http://jellyfin:8096/jellyfin/;

        proxy_pass_request_headers on;

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $http_host;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;

        # Disable buffering when the nginx proxy gets very resource heavy upon streaming
        proxy_buffering off;
    }
```
- Restart containers

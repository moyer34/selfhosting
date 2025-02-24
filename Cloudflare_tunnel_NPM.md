# Remote Access to Local Services (Cloudflare Tunnel - NPM Setup)

This document describes steps to access locally-hosted services from a public URL. This does not require local ports or exposure of the local network's public IP address. It relies on the following components:
- **Cloudflare Tunnel** - When configured, it directs a subdomain for a domain manager in Cloudflare into the local network, without opening ports, to a locally hosted service using …
- **Cloudflared docker container** - this container takes the auth token for the tunnel and is the end point inside the local network
- **Nginx Proxy Manager**

End result:
- A publicly accessible url can direct to a local service
- No firewall ports are opened
- Cloudflare can only see nginx proxy manager
- Services can only see nginx proxy manager

Important: Cloudflare will deliver SSL encryption from the public internet, and the tunnel itself is encrypted, but with this setup traffic in the internal network is NOT encrypted. Cloudflare also can see everything.

## Prerequisites
- Understanding of Docker, Docker Compose, and defining networks in Docker Compose files;
- Sign up for a Cloudflare account;
- Transfer a top level domain name to Cloudflare’s name server.

## 1-Set up Cloudflare Tunnel
- select Zero Trust from sidebar in Cloudflare
- Select Networks -> Tunnels
- Create a new tunnel. Select "Cloudflared" option
- Name this tunnel and save it
- Back on the page listing all tunnels in the account, select the 3-dot menu for the new tunnel and press Configure
- Under "Install and run," choose docker environment and copy the command somewhere temporarily. Do not run the command, we just need the token to insert into the docker compose environment file

## Set up service stack using Docker Compose
We will set up a service stack using docker compose. There are alternatives to this approach, but I’ve had success including the service I want to point to, NPM, and Cloudflared in the same docker compose file. This makes the docker network config more intuitive.

Sample docker-compose.yml:
```yaml
version: "3"
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    ports:
      # HTTP/S Ports only enabled inside docker network for security
      #     - ${HOST_PORT_HTTP}:80
      #     - ${HOST_PORT_HTTPS}:443
      - ${HOST_PORT_ADMIN}:81
    volumes:
      - ${DATA_DIR_NPM}/data:/data   #I choose to do bind mounts but you can use vanilla docker volumes.
      - ${DATA_DIR_NPM}/letsencrypt:/etc/letsencrypt
    restart: always
    networks:
      external:
        ipv4_address: 172.XX.0.4 # in this example, NPM is set to .4 as a static IP. In Cloudflare, you will redirect the URL to this IP address
      npminternal: null
  tunnel:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-tunnel
    command: tunnel --no-autoupdate run --token ${CF_TUNNEL_TOKEN}
    restart: unless-stopped
    networks:
      - external
  #-Services - arbitrarily chose the app Chatpad--------
  #--The important thing is that all local services accessed by the tunnel via Nginx Proxy Manager are part of the "npminternal" network
  #--AND have a container_name defined that can be referenced in Nginx Proxy Manager 
  chatpad:
    image: ghcr.io/deiucanta/chatpad:latest
    container_name: chatpad
    ports:
      - 4080:80 # In NPM, you will 
    restart: unless-stopped
    networks:
      - npminternal

#-Global-----------
networks:
  external:
    ipam:
      driver: default
      config:
        - subnet: 172.XX.0.0/16 #Replace XX with a valid/available docker network octet.
  npminternal:
    ipam:
      driver: default
      config:
        - subnet: 172.YY.0.0/16 #Replace YY with a valid/available docker network octet, different from the one above.

```

Sample environment file:

```yaml
HOST_PORT_ADMIN=<Internal port to use for Nginx Proxy Manager admin>
DATA_DIR_NPM=<Local docker volume mapped directory. Just use Docker volumes as an alternative.>
CF_TUNNEL_TOKEN=<INSERT SUPER LONG CLOUDFLARE TUNNEL TOKEN WITH NO BRACKETS>
```

### Notes on the Docker networking setup
The docker networks are set up intentionally to isolate services.
- external - Tunnel -> NPM. Allows cloudflared (and therefore the tunnel) to see only NPM
- npm_internal - NPM -> Services. Allows npm and services to see each other.

Thus NPM is serving its purpose as a reverse proxy, taking subdomain requests from the tunnel and shuffling them to the correct internal service ip and port. You can also skip NPM and send the tunnel directly to the end services, but I like this bit of complexity since it isolates the surface accessible from the tunnel. Note that in external, NPM is reserved a static ip in the docker network. This is so that the cloudflare tunnel can always point the subdomain to the same IP address. I had instances where after updates the NPM IP address changed on me, causing a 404 error.

### Configuring Nginx Proxy Manager
For each service, set up a proxy host in the Nginx Proxy Manager admin interface as follows:
- *Domain Names:* enter the full domain as you would in a browser. NPM will see this coming in from Cloudflare 
- *Forward Hostname / IP:* point to the IP of the service (or better yet the service container_name). In the example docker compose, "chatpad".
- *Forward Port:* enter the INTERNAL port for the service. This is because NPM and the service are part of the same NPM_internal docker network. In the example docker compose, 80 (not 4080). 
- *Scheme (http/https):* set http not https. This is an important caveat: Cloudflare will deliver SSL encryption from the public internet, and the tunnel itself is encrypted, but with this setup traffic in the internal network is NOT encrypted. Cloudflare also can see everything.

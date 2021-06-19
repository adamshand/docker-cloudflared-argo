
See original post at: https://www.reddit.com/r/selfhosted/comments/o39ok6/has_anyone_got_argo_tunnels_to_multiple_endpoints/

> UPDATE: I now have this working. When you are proxying a single host the `cloudflared` daemon creates the DNS entries for you.  When you are using a config file you must create the DNS entries manually. You can either do this at cloudflare.com or use the command line:  
>  
> `# cloudflared tunnel route dns xxxxxxx-1356-xxxx-af44-237e463731 music.example.nz`  
>  
> Here is the [documentation I missed](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/routing-to-tunnel/dns).  Hopefully my mistake means that somebody else gets it easy. :-)  

> UPDATE 2: SSH access is a little more complicated as there is no port 22 available on CloudFlare and you can't get to the Docker host via `ssh://localhsot:22` from inside a container.  There are actually [good docs on how to do this](https://developers.cloudflare.com/cloudflare-one/tutorials/ssh), I just missed them. 

I have the sad reality of being stuck behind a CGNAT setup at home, and so CloudFlare's recent free offering of Argo tunnels is pretty cool.

I've set it up to run as a Docker container and [followed the documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/configuration/ingress).  When I bring the container up I don't get any errors but the DNS entries never get created.

Has anyone got Argo tunnels to multiple endpoints working?

Here's my setup:

## docker-compose.yml (multiple sites)

    ---
    version: '3'
    services:
      argo:
        image: cloudflare/cloudflared:2021.5.10-amd64
        container_name: argo
        restart: unless-stopped
        volumes:
          - ./conf:/etc/cloudflared
        command: tunnel --config /etc/cloudflared/config.yml run
        environment:
          - TUNNEL_ORIGIN_CERT=/etc/cloudflared/cert.pem
          - TUNNEL_LOGLEVEL=info
        networks:
          - argo
    
    networks:
      argo:
        external: true

## config.yml

    tunnel: xxxxxxx-1356-xxxx-af44-237e463731
    credentials-file: /etc/cloudflared/xxxxxxx-1356-xxxx-af44-237e463731.json
    
    ingress:
      - hostname: music.example.nz
        service: http://navidrome:80
      - hostname: home.example.nz
        service: ssh://10.11.12.1:22
      - service: http_status:404

## ~/.ssh/config
```
Host home home.example.nz
  Hostname home.example.nz
  ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
```

In case it's useful to anyone else, I've got it working for a single site (without a config.yml) using the below docker-compose file:

## docker-compose.yml (single site)

    ---
    version: '3'
    services:
      argo:
        image: cloudflare/cloudflared:2021.5.10-amd64
        container_name: argo
        restart: unless-stopped
        volumes:
          # run: cloudflared tunnel login
          # then copy the contents of ~/.cloudflared to ./conf
          - ./conf:/etc/cloudflared
        command: tunnel --no-autoupdate --hostname music.example.nz --url http://navidrome
        environment:
          - TUNNEL_ORIGIN_CERT=/etc/cloudflared/cert.pem
          - TUNNEL_LOGLEVEL=info  # trace, debug, info, warn, error, fatal, panic
        networks:
          - argo
    
    # make sure that this container can talk to your destination container
    # (in this case navidrom) by either putting them all in a single docker-compose.yml 
    # or adding the argo network to the destination containers config as well. 
    networks:
      argo:
        external: true


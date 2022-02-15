# docker-traefik
Docker Compose for Traefik on Docker

--- 
# Setup Static And Dynamic Configuration With Traefik On Docker

Lets rewrite our docker-compose.yml script for Traefik. If you already have the service up and running you will need to unload it with docker-compose down
```
vi docker-compose.yml
```
Replace the existing Traefik script with this.
```
version: "3.3"
services:
  traefik:
    image: traefik:v2.4
    command:
      - --configFile=/etc/traefik/traefik.yml
    ports:
      - 80:80
      - 443:443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/traefik:/etc/traefik
```
Commands and labels are now serving with traefik.yml located in /etc/traefik. Let’s create those files.
```
sudo mkdir -p /etc/traefik && cd /etc/traefik
```
Create traefik.yml, routes.yml and acme.json
```
sudo touch /etc/traefik/traefik.yml
sudo touch /etc/traefik/routes.yml
sudo touch /etc/traefik/acme.json
```
- traefik.yml is the static configuration file which contains elements that set up connections to providers and define the entrypoints Traefik will listen to (these elements don’t change often).
- routes.yml is the dynamic configuration contains everything that defines how the requests are handled by your system. This configuration can change and is seamlessly hot-reloaded, without any request interruption or connection loss.
- acme.json stores your Let’s Encrypt certificates

# Static Configuration
Edit traefik.yml and paste the following script:

```
vi /etc/traefik/traefik.yml
```
```
api:
  dashboard: true                             

certificatesResolvers:
  le:
    acme:
      email: "admin@localhost.com"  
      storage: "/etc/traefik/acme.json"    
      tlsChallenge: {}

entryPoints:
  web:
    address: ":80"                            
    http:
      redirections:                           
        entryPoint:
          to: "websecure"                     
          scheme: "https"                     
  websecure:
    address: ":443"                           

global:
  checknewversion: true                       
  sendanonymoususage: true                    

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"   
    exposedByDefault: false                   
    network: "traefik-public"                 
  file:
    filename: "/etc/traefik/routes.yml"       
    watch: true     
```
Remember to change email address for Let’s Encrypt ACME!
Finally, we will need to change the permission setting of our acme.json file
```
sudo chmod 600 /etc/traefik/acme.json
```
# Dynamic Configuration

From our static configuration, we have defined our dynamic end point at /etc/traefik/routes.yml. Because dynamic configuration changes often and are dependent on your existing container apps, we will provide a base template for dynamic routing:

Edit /etc/traefik/routes.yml
```
vi /etc/traefik/routes.yml
```
```
http:
  routers:
    app-http:
      entryPoints:
      - http
      middlewares:
      - https-redirect
      rule: "Host(`example.com`)"
      service: app
    app-https:
      entryPoints:
      - https
      rule: "Host(`example.com`)"
      service: app
      tls:
        certResolver: le

  services:
    app:
      loadBalancer:
        servers:
          - url: "http://proxy"

  middlewares:
    https-redirect:
      redirectScheme:
        scheme: https
        permanent: true
```

Change example.com to your FQDN (A Record)
Change proxy to your ingress
The rest should handle itself include automated Let’s Encrypt and https redirection middleware.

Once you are ready resume your services with:

```
docker-compose up -d
```

# References
All available static environment variables can be found [here](https://doc.traefik.io/traefik/reference/static-configuration/file/).

All available dynamic environment variables can be found [here](https://doc.traefik.io/traefik/reference/dynamic-configuration/file/).

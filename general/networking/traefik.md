## Traefik (with docker-compose)

Traefik is a modern HTTP reverse proxy and load balancer that makes deploying microservices easy. Traefik integrates with your existing infrastructure components (Docker, Swarm mode, Kubernetes, Marathon, Consul, Etcd, Rancher, Amazon ECS, ...) and configures itself automatically and dynamically. Pointing Traefik at your orchestrator should be the only configuration step you need. (https://traefik.io/). 

Create these 3 files in the SAME directory (or change their paths in the volume section) : docker-compose.yml, traefik.toml and acme.json.

This configuration is A+ (SSLlabs)

docker-compose.yml:

```
version: '3.5'

services:
  traefik:
    container_name: traefik
    image: traefik:v1.7
    networks:
      - traefik
    ports:
      - 80:80
      - 443:443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./traefik.toml:/traefik.toml
      - ./acme.json:/acme.json
    labels:
      traefik.enable: "true"
      traefik.backend: traefik
      traefik.docker.network: traefik
      traefik.port: 8080
      traefik.frontend.rule: Host:traefik.example.com,
      traefik.frontend.entryPoints: https
      traefik.frontend.passHostHeader: "true"
      traefik.frontend.headers.SSLForceHost: "true"
      traefik.frontend.headers.SSLHost: traefik.example.com
      traefik.frontend.headers.SSLRedirect: "true"
      traefik.frontend.headers.browserXSSFilter: "true"
      traefik.frontend.headers.contentTypeNosniff: "true"
      traefik.frontend.headers.forceSTSHeader: "true"
      traefik.frontend.headers.STSSeconds: 315360000
      traefik.frontend.headers.STSIncludeSubdomains: "true"
      traefik.frontend.headers.STSPreload: "true"
      traefik.frontend.headers.customResponseHeaders: X-Robots-Tag:noindex,nofollow,nosnippet,noarchive,notranslate,noimageindex
      traefik.frontend.headers.frameDeny: "true"
      traefik.frontend.headers.customFrameOptionsValue: 'allow-from https://example.com'
    restart: unless-stopped

  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    network_mode: "host"
    volumes:
      - /path/to/config:/config
      - /path/to/cache:/cache
      - /path/to/media:/media
    restart: unless-stopped

networks:
  traefik:
    name: traefik
```

This toml file can't support environment variables, ensure you don't attempt to use variables.

traefik.toml:
 
```
logLevel = "WARN"
defaultEntryPoints = ["http", "https"]

[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
    minVersion = "VersionTLS12"
    cipherSuites = [
      "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384",
      "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
      "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305",
      "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305",
      "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
      "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
      "TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256",
      "TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256"
    ]

[retry]

[api]

[acme]
acmeLogging = true
email = "user@example.com"
storage = "acme.json"
entryPoint = "https"
  [acme.dnsChallenge]
    provider = "provider"
    delayBeforeCheck = "60"

[[acme.domains]]
  main = "*.example.com"
  
[docker]
domain = "example.com"
network = "traefik"
exposedbydefault = false

[file]
[backends]
  [backends.backend-jellyfin]
    [backends.backend-jellyfin.servers]
      [backends.backend-jellyfin.servers.server-1]
        url = "http://172.17.0.1:8096"
[frontends]
  [frontends.jellyfin]
    backend = "backend-jellyfin"
    passHostHeader = true
    [frontends.jellyfin.routes]
      [frontends.jellyfin.routes.route-jellyfin-ext]
        rule = "Host:jellyfin.example.com"
    [frontends.jellyfin.headers]
      SSLRedirect = true
      SSLHost = "jellyfin.example.com"
      SSLForceHost = true
      STSSeconds = 315360000
      STSIncludeSubdomains = true
      STSPreload = true
      forceSTSHeader = true
      frameDeny = true
      contentTypeNosniff = true
      browserXSSFilter = true
      customResponseHeaders = "X-Robots-Tag:noindex,nofollow,nosnippet,noarchive,notranslate,noimageindex"
      customFrameOptionsValue = "allow-from https://example.com"
```

Finally, create an empty acme.json.

```bash
touch acme.json
chmod 600 acme.json
```

IMPORTANT ! Change example.com to your domain / subdomain name, and change the mail of the acme (user@example.com in traefik.toml). Let's Encrypt does not require a valid email address however example.com will be flagged as not being a proper email address.

Launch your Traefik/Jellyfin services : `docker-compose up -d`

Congratulations, your stack with Traefik and Jellyfin is running!

> [!NOTE]
> Due to a [bug](https://github.com/containous/traefik/issues/5559) in Traefik, you cannot dynamically route to containers when network_mode=host, so we have created a static route to the docker host (172.17.0.1:8096) in `traefik.toml`. Using host networking (or macvlan) is required to use DLNA or an HdHomeRun as it supports multicast networking.

Go to jellyfin.example.com (in this case), and your jellyfin is running with HTTPS (AES 256).
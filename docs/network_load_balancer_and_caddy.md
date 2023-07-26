We will use AWS Copilot to build a Network Load Balancer. It cannot be Application Load Balancer or Classic Load Balancer because we need the encrypted traffic to flow through the load balancer to Caddy instance. And then Caddy will be able to do its work.

Here's the manifest for building Network Load Balancer using AWS Copilot:

``` yaml title="manifest.yml" linenums="1" hl_lines="18 19 20 21 22"
# The manifest for the "caddy-webserver" service.
# Read the full specification for the "Load Balanced Web Service" type at:
#  https://aws.github.io/copilot-cli/docs/manifest/lb-web-service/

# Your service name will be used in naming your resources like log groups, ECS services, etc.
name: nlb
type: Load Balanced Web Service

# Distribute traffic to your service.

# Configuration for your containers and service.
image:
  # Docker build arguments. For additional overrides: https://aws.github.io/copilot-cli/docs/manifest/lb-web-service/#image-build
  build: Dockerfile
  # Port exposed through your container to route traffic to it.
  port: 443

http: false
nlb:
  port: 443/tcp
  additional_listeners:  
    - port: 80/tcp  


cpu: 256       # Number of CPU units for the task.
memory: 512    # Amount of memory in MiB used by the task.
platform: linux/x86_64  # See https://aws.github.io/copilot-cli/docs/manifest/lb-web-service/#platform
count: 3       # Number of tasks that should be running in your service.
exec: true     # Enable running commands in your container.
network:
  connect: true # Enable Service Connect for intra-environment traffic between services.
```

## Dockerfile
``` dockerfile title="Dockerfile" linenums="1" hl_lines="12 13"
FROM amazonlinux:2

ARG CADDY_VERSION=v2.6.4
ARG OS=linux

# Install dependencies
RUN yum update -y && \
  yum install golang -y

RUN GOBIN=/usr/local/bin/ go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest

RUN mkdir -p /caddy-build && \
  GOOS=${OS} xcaddy build ${CADDY_VERSION} --with github.com/gamalan/caddy-tlsredis --output /caddy-build/caddy

COPY Caddyfile /etc/caddy/Caddyfile

EXPOSE 80
EXPOSE 443

CMD ["/caddy-build/caddy", "run", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]
```

Line 12, 13: Building caddy with Redis. So certs can be stored in Redis, and retrieved by Caddy instance.

## Caddyfile
Caddyfile is Caddy's config file.
``` title="Caddyfile" linenums="1" hl_lines="4 12 13 14 16 17 18 19 20"
{
    debug
    on_demand_tls {
        ask https://portfolio-starter-kit-beta-six.vercel.app/api/tls_authoriser
        
        burst 5
        interval 2m
    }
}

https:// {
    tls {
        on_demand
    }

    reverse_proxy https://portfolio-starter-kit-beta-six.vercel.app {
        header_down Strict-Transport-Security max-age=31536000
        header_up X-Real-IP {remote}
        header_up Host {upstream_hostport}
    }
}
```
Line 4: Caddy will ask the specified endpoint for authorisation, i.e. can Caddy fetch and install a cert for a custom domain? The endpoint must return 200 for allow, and any other statuses (eg 403) for deny. The endpoint, in this case, is a webserver, it can be of other forms like Lambda, as long as it returns 200.

Example of an endpoint:
``` js title="/pages/api/tls_authoriser.js"
export default function handler(req, res) {
  res.setHeader('Cache-Control', 'no-store, max-age=0')
  res.status(200).json({})
}
```
Line 12-14: Cernts are fetched and installed on demand.

Line 16-20: Once a cert is fetched and installed, proxy traffic to original domain.
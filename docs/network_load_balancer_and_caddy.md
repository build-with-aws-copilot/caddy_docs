We will use AWS Copilot to build a Network Load Balancer. It cannot be Application Load Balancer or Classic Load Balancer because we need the encrypted traffic to flow through the load balancer to Caddy instances. And then Caddy will be able to do its work.

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

Line 12, 13: Building caddy with Redis. So certs can be stored in Redis, and retrieved by multiple Caddy instances.

## Caddyfile
Caddyfile is Caddy's config file.
``` title="Caddyfile" linenums="1" hl_lines="4 12 13 14 16 17 18"
{
    debug
    on_demand_tls {
        ask https://tictactoeonrails.tamsui.xyz/tls_check
        
        burst 5
        interval 2m
    }
}

https:// {
    tls {
        on_demand
    }

    reverse_proxy https://tictactoeonrails.tamsui.xyz {
        header_up X-Real-IP {remote}
    }
}
```
Line 4: Caddy will ask the specified endpoint for authorisation, i.e. can Caddy fetch and install a cert for a custom domain? The endpoint must return 200 for allow, and any other statuses (eg 403) for deny. The endpoint, in this case, is a webserver, it can be of other forms like Lambda, as long as it returns 200.

Example of an endpoint:
``` rb title="tls_check_controller.rb"
class TlsCheckController < ActionController::Base
  def check
    requested_domain = params[:domain] # ?domain=www.clientdomain.com

    return head :ok if requested_domain.owner.paid? # only issue cert when client has paid

    head :forbidden # if client hasn't paid, forbid the generation of cert
  end
end
```
Line 12-14: Cernts are fetched and installed on demand.

Line 16-18: Once a cert is fetched and installed, proxy traffic to your domain.
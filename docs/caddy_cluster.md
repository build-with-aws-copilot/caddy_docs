In previous chapter, we provisioned just a single Caddy instance, and the TLS certificate is stored on the instance. In this chapter we will build a Caddy cluster so each Caddy instance is load-balanced. We will also build a Redis server, TLS certificates will be stored on Redis and can be retrieved by Caddy cluster.

``` yaml title="manifest.yml" linenums="1" hl_lines="28"
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
line 28: change task count from 1 to 3.

## Adding a Redis server
``` yaml title="copilot/caddy/addons/template.yml" linenums="1"
Parameters:
  App:
    Type: String
    Description: Your application's name.
  Env:
    Type: String
    Description: The environment name your service, job, or workflow is being deployed to.
  Name:
    Type: String
    Description: The name of the service, job, or workflow being deployed.

Resources:
  # Subnet group to control where the Redis gets placed
  RedisSubnetGroup:
    Type: AWS::ElastiCache::SubnetGroup
    Properties:
      Description: Group of subnets to place Redis into
      SubnetIds: !Split [ ',', { 'Fn::ImportValue': !Sub '${App}-${Env}-PrivateSubnets' } ]
  
  # Security group to add the Redis cluster to the VPC,
  # and to allow the Fargate containers to talk to Redis on port 6379
  RedisSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Redis Security Group"
      VpcId: { 'Fn::ImportValue': !Sub '${App}-${Env}-VpcId' }
  
  # Enable ingress from other ECS services created within the environment.
  RedisIngress:
    Type: AWS::EC2::SecurityGroupIngress
    Properties:
      Description: Ingress from Fargate containers
      GroupId: !Ref 'RedisSecurityGroup'
      IpProtocol: tcp
      FromPort: 6379
      ToPort: 6379
      SourceSecurityGroupId: { 'Fn::ImportValue': !Sub '${App}-${Env}-EnvironmentSecurityGroup' }

  # The cluster itself.
  Redis:
    Type: AWS::ElastiCache::CacheCluster
    Properties:
      Engine: redis
      CacheNodeType: cache.t2.micro
      NumCacheNodes: 1
      CacheSubnetGroupName: !Ref 'RedisSubnetGroup'
      VpcSecurityGroupIds:
        - !GetAtt 'RedisSecurityGroup.GroupId'

  # Redis endpoint stored in SSM so that other services can retrieve the endpoint.
  RedisEndpointAddressParam:
    Type: AWS::SSM::Parameter
    Properties:
      Name: !Sub '/${App}/${Env}/${Name}/redis'   # Other services can retrieve the endpoint from this path.
      Type: String
      Value: !GetAtt 'Redis.RedisEndpoint.Address'

Outputs:
  RedisEndpoint:
    Description: The endpoint of the redis cluster
    Value: !GetAtt 'Redis.RedisEndpoint.Address'
    Export:
      Name: RedisEndpoint
```

## Adding environment variables to Network Load Balancer Manifest
``` yaml title="copilot/nlb/manifest.yml" linenums="1"
variables:
  CADDY_CLUSTERING_REDIS_HOST:
    from_cfn: RedisEndpoint
  CADDY_CLUSTERING_REDIS_TLS: false
  CADDY_CLUSTERING_REDIS_TLS_INSECURE: true
```

## Updating Caddyfile to store TLS certs in Redis
```  linenums="1" hl_lines="11 12 13"
{
    debug
    on_demand_tls {
        ask https://portfolio-starter-kit-beta-six.vercel.app/api/tls_authoriser
        
        burst 5
        interval 2m
    }

    storage redis {
        host {$CADDY_CLUSTERING_REDIS_HOST}
        tls_enabled {$CADDY_CLUSTERING_REDIS_TLS}
        tls_insecure {$CADDY_CLUSTERING_REDIS_TLS_INSECURE}
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
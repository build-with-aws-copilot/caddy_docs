After Network Load Balancer is provisioned, it will have a domain name, for example
```
caddy-Publi-JRJUD8MD0RLE-671e55804baa69eb.elb.us-west-2.amazonaws.com
```
You only need to give this domain name to your client, and he/she will create a CNAME record in their DNS. For example:

<img width="374" alt="Screen Shot 2023-08-11 at 9 03 27 am" src="https://github.com/build-with-aws-copilot/caddy/assets/129698988/170721e2-bf8d-4719-8322-f7fcc6b3aafc">

And your client's domain (c21.awscustomdomainmanager.xyz) will have access to your service.
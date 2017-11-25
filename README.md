## kickerd-nginx

[![Build Status][travis-image]][travis-url]

Adds [kickerd](https://github.com/arobson/kickerd) hosting for NGiNX so that you can source SSL certs from etcd and have your ingress containers rolling-restart when the certs are updated.

### Motivation
Use NGiNX between Cloud Provider's ELB and Kubernetes hosted services for:
 * SSL termination
 * static resource caching
 * compression
 * other stuff NGiNX is good at ...

Instead of producing static config files that map service IPs to upstream blocks, use Kubernetes service names. Multiple Pod IPs can be assigned and these assignments can change over time. NGiNX only resolves these DNS names at start-up.

## Example `.kickerd.toml`

The following example shows how you would map a `.kickerd.toml` file onto the container so that when it starts, kickerd will pull the keys on the right into the file paths on the left before starting the NGiNX process.

If anything changes the key contents, kickerd will acquire a lock for the process in etcd and then restart the NGiNX process within the scope of a locked etcd key, so that all your NGiNX containers don't disappear at once.

```toml
name = "nginx"
description = "nginx service"
start = "nginx -g daemon off"

[file]
  '/etc/nginx/config/htpasswd' = "{{site-htpasswd}}"
  '/etc/nginx/cert/site1.cert' = "{{site1-cert}}"
  '/etc/nginx/cert/site1.key' = "{{site1-key}}"
  '/etc/nginx/cert/site2.cert' = "{{site2-cert}}"
  '/etc/nginx/cert/site2.key' = "{{site2-key}}"
```

## Example NGiNX Location Block

Assuming you can allow round-robin balancing from NGiNX to your services, you can use the following NGiNX configuration style per location block (things to replace are in CAPS):

```
  server {
    listen    443 ssl;
    listen    [::]:443 ssl;
    root      /usr/share/nginx/html;

    ssl on;
    ssl_certificate       "/CERT/PATH";
    ssl_certificate_key   "/KEY/PATH";

    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers HIGH:SEED:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!RSAPSK:!aDH:!aECDH:!EDH-DSS-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA:!SRP;
    ssl_prefer_server_ciphers on;

    server_name   ~^SUBDOMAIN[.].*$;

    location / {
      resolver            kube-dns.kube-system valid=1s;
      set $server         NAME.NAMESPACE.svc.cluster.local:PORT;
      rewrite             ^/(.*) /$1 break;
      proxy_pass          http://$server;
      proxy_set_header    Host $host;
      proxy_set_header    X-Real-IP $remote_addr;
      proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header    X-Forwarded-Proto $scheme;
    }
  }
```

## DEPRECATED - jdomain

A previous version of this image included [jdomain](https://www.nginx.com/resources/wiki/modules/domain_resolve/) module. In newer versions of NGiNX, this no longer works. It was originally included to provide an alternative means of resolving back-ends so that when service IPs changed, NGiNX would still resolve them.


## Acknowledgements

### NGiNX Docker Maintainers

The image is a modification of the official NGiNX image. Most notable deviations are the inclusion of Node and kickerd as well as use of alpine 3.6 in order to remove the presence of the musl vulnerability, [CVE-2016-8859](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-8859).

### Contributions

Thanks to [Oliver Poitrey's](https://github.com/rs) for his PRs/improvements to this module.

[travis-url]: https://travis-ci.org/npm-wharf/kickerd-nginx
[travis-image]: https://travis-ci.org/npm-wharf/kickerd-nginx.svg?branch=master
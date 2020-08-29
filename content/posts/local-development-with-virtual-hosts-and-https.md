+++
keywords = [
  'Docker',
  'Nginx',
  'Development',
  'Vhosts',
  'DNS',
  'LetsEncrypt',
  'SSL',
  'HTTPS',
  'Web',
  'OSX',
  'Mac',
  'Dig',
]
categories = [
  'Development',
  'Web',
  'Docker'
]
title = "Local development with virtual hosts and HTTPS"
date = "2019-02-22T8:00:00Z"
description = "Using docker to provide virtual hosts and HTTPS to your local applications"

+++

<center>![](/images/local-development-with-virtual-hosts-and-https/diagram.png)</center>
 
When doing development locally it might be necessary to access the application(s) using a virtual host (vhost) and/or HTTPS. This post describes an approach to achieving this on OSX using Docker, which avoids creating a large mess on your computer.

## Domain Name Systems (DNS)

Domain Name Systems (DNS) are often referred to as the phonebook of the internet. People access information online through domain names, like ‘amazon.com’ or ‘rte.ie’. DNS is responsible for translating a domain name to an IP address.

When an HTTP resource is accessed using DNS, an additional header containing the domain name is provided to the HTTP server. The HTTP server uses this for routing the request to the correct vhost.

In order to have functioning vhosts, DNS is required. [Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) is a popular DNS server, it can be configured to resolve a set of specified domains to a specified IP address. Using docker a dnsmasq server can easily be created:

```
$ docker run -it -p 53:53/udp --cap-add=NET_ADMIN andyshinn/dnsmasq:latest --log-queries --log-facility=- --address=/dev.ianduffy.ie/127.0.0.1
```

The provided command line arguments instruct dnsmasq to resolve all requests for dev.ianduffy.ie and *.dev.ianduffy.ie to 127.0.0.1. Using dig - a DNS lookup utility this configuration can be validated:

```
$ dig @127.0.0.1 dev.ianduffy.ie +short
127.0.0.1
```

Additionally, all wildcards of dev.ianduffy.ie also resolve:

```
$ dig @127.0.0.1 random-string.dev.ianduffy.ie +short
127.0.0.1
```

While this works, the OSX is not configured to use this DNS server for it’s lookups so attempting to resolve dev.ianduffy.ie in the browser will fail. It's possible to specify nameservers for a specific domain name, this can be achieved by creating a file within /etc/resolver with a filename that matches the domain.

For example, if /etc/resolver/dev.ianduffy.ie is created with the following contents:

```
nameserver 127.0.0.1
```

Now all queries to dev.ianduffy.ie will be resolved by doing a lookup against the DNS server at 127.0.0.1.

## HTTP

A HTTP server will be required for routing the requests based on a vhost and supplying HTTPS. Nginx is a good fit for this, even-more-so as [Jason Wilder](https://www.linkedin.com/in/jason-wilder-94549/) from Microsoft has created a  [container image](https://github.com/jwilder/nginx-proxy) that exposes the nginx reverse proxy functionality via environment variables.

[nginx-proxy](https://github.com/jwilder/nginx-proxy) can be started using the following:

```
$ docker run -it -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro jwilder/nginx-proxy
```

[nginx-proxy](https://github.com/jwilder/nginx-proxy) will look for `VIRTUAL_HOST` environment variables on other docker containers and route to them accordingly. To demonstrate this, a container running [httpbin](https://httpbin.org) which provides data for debugging HTTP requests can be created, with a VIRTUAL_HOST environment variable specified.

```
$ docker run -e VIRTUAL_HOST=httpbin.dev.ianduffy.ie kennethreitz/httpbin
```

This service can now be accessed via httpbin.dev.ianduffy.ie. Alternatively, if you do not have the dnsmasq service from earlier running, the service can be accessed by passing the header "host" with value "httpbin.dev.ianduffy.ie". This can be tested with an HTTP client like [curl](https://curl.haxx.se/)

```
$ curl http://httpbin.dev.ianduffy.ie/headers
{
  "headers": {
    "Accept": "*/*",
    "Connection": "close",
    "Host": "httpbin.dev.ianduffy.ie",
    "User-Agent": "curl/7.54.0"
  }
}
```

```
$ curl -H "host: httpbin.dev.ianduffy.ie" http://127.0.0.1/headers
{
  "headers": {
    "Accept": "*/*",
    "Connection": "close",
    "Host": "httpbin.dev.ianduffy.ie",
    "User-Agent": "curl/7.54.0"
  }
}
```

## HTTPS

In some scenarios, HTTPS might be required. [mkcert](https://github.com/FiloSottile/mkcert) provides locally trusted SSL certificates and automatically OSX, Linux, and Windows system stores along with Firefox, Chrome and Java.


With [mkcert](https://github.com/FiloSottile/mkcert) installed, certificates can be generated with the following command:

```
$ mkdir certs
$ mkcert -cert-file certs/dev.ianduffy.ie.crt -key-file certs/dev.ianduffy.ie.key -install dev.ianduffy.ie *.dev.ianduffy.ie
```

By mounting these certificates, as a volume on a container running nginx-proxy HTTPS will be enabled.

```
$ docker run -it -p 80:80 -p 443:443 -v $(pwd)/certs:/etc/nginx/certs -v /var/run/docker.sock:/tmp/docker.sock:ro jwilder/nginx-proxy
```

Executing `curl` to https://httpbin.dev.ianduffy.ie will now respond successfully. Additionally, the `-v` flag can be specified to tell curl to be verbose and it will display information about the SSL certificate.

```
$ curl -v https://httpbin.dev.ianduffy.ie/headers
```

## Placing a vhost in front of a local application

Most developers will want to run their application on the host and continue with the standard development workflow. To accommodate for this, a container that forwards traffic to a host port can be created.

[socat](http://www.dest-unreach.org/socat/doc/README) is perfect for this use case, it enables us to specify a source IP address and port and a destination IP address and port. In the example below, all traffic for test.dev.ianduffy.ie will be forwarded to the docker host on port 9000.

```
$ docker run -it --expose 80 -e VIRTUAL_HOST=test.dev.ianduffy.ie alpine/socat tcp-listen:80,fork,reuseaddr tcp-connect:host.docker.internal:9000
```
 
 
## Bring it all together with docker-compose

All of the containers described above can be brought together in a single docker compose file. This provides ease of bringing the system up with a single command.

```
nginx-proxy:
  container_name: nginx-proxy
  image: jwilder/nginx-proxy
  ports:
    - 80:80
    - 443:443
  volumes:
    - /var/run/docker.sock:/tmp/docker.sock:ro
    - ./certs:/etc/nginx/certs:ro
dnsmasq:
  container_name: dnsmasq
  image: andyshinn/dnsmasq:latest
  command: --log-queries --log-facility=- --address=/dev.ianduffy.ie/127.0.0.1
  ports:
    - 53:53
    - 53:53/udp
  cap_add:
    - NET_ADMIN
socat:
  image: alpine/socat
  command: tcp-listen:80,fork,reuseaddr tcp-connect:host.docker.internal:9000
  environment:
    - VIRTUAL_HOST=dev.ianduffy.ie
    - VIRTUAL_PORT=80
  expose:
    - 80
```

This enables VHosts and HTTPS for applications running on the host without creating too much of a mess as its all contained within Docker.

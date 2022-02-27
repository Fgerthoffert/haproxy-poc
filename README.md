# Using HAProxy to expose resources in a Docker network

## The Issue

At Jahia our teams are developing a complex platform composed of multiple resources, some operating in cluster using the same TCP/ports. 

Recent issues encountered in particular cluster situations highlighted some challenges in reproducing cluster issues in a dev environment, implementing tests to prevent re-occurence of these issues and being able to easily run these tests in both a local dev environment and a remote CI platform.

In other words, how do you address resources directly without getting into a Docker port binding nightmare ?

### Exposing the problem

Let's take the architecture below as an example, you have an environment composed of:

* 3 Apps with a public interface accessible on port 8080. These apps are operating as a cluster and talk to eachother (the actual inter-node communication is not relevant for our example). 
* 1 reverse proxy, which will balance public request between the 3 nodes
* 1 testing app, which will go through HAProxy for validating the tests, but who will also need, in some situations to perform an operation on one single node.

```
┌────────────────────────────────────────────────┐
│                                                │
│    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│    │         │    │         │    │         │   │
│    │  APP_1  │◄──►│  APP_2  │◄──►│  APP_3  │   │
│    │         │    │         │    │         │   │
│    └─────────┘    └─────────┘    └─────────┘   │
│        80             80             80        │
│         ▲              ▲             ▲  ▲      │
│         │              │             │  │      │
│         ├──────────────┴─────────────┘  │      │
│         │                               │      │
│    ┌────┴──────┐               ┌────────┴──┐   │
│    │           │               │           │   │
│    │  Haproxy  │80 ◄───────────┤  Tests    │   │
│    │           │               │           │   │
│    └───────────┘               └───────────┘   │
│                                                │
│                                 Docker Network │
└────────────────────────────────────────────────┘
```
If all resources are running as independant Docker containers, or in different machines/VMs, it all works well. If using Docker, each container is considered as its own host, and can expose the port of its choice.

But what about developing the tests, how do you execute them outside of Docker and still be able to address any of the ressources individually?

Like this:

```
┌────────────────────────────────────────────────┐
│                                                │
│    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│    │         │    │         │    │         │   │
│    │  APP_1  │◄──►│  APP_2  │◄──►│  APP_3  │   │
│    │         │    │         │    │         │   │
│    └─────────┘    └─────────┘    └─────────┘   │
│        80             80             80        │           ┌───────────┐
│         ▲              ▲             ▲         │           │           │
│         │              │             │         │◄──────────┤  Tests    │
│         ├──────────────┴─────────────┘         │           │     (dev) │
│         │                                      │           └───────────┘
│    ┌────┴──────┐                               │
│    │           │                               │
│    │  Haproxy  │ 80                            │
│    │           │                               │
│    └───────────┘                               │
│                                 Docker Network │
└────────────────────────────────────────────────┘
```

### The port binding approach

The traditional solution to accessing resources running in Docker is to use port binding, doing something like this:

```
HAProxy:80 -> localhost:80
APP_1:80 -> localhost:81
APP_2:80 -> localhost:82
APP_3:80 -> localhost:83
```

Our `docker-compose.yml` file would look like this:

```yaml
version: '3.8'

networks:
  mynetwork:
    name: mynetwork

services:
  app_1:
    image: nginxdemos/hello
    ports:
      - 81:80
    networks:
      - mynetwork

  app_2:
    image: nginxdemos/hello
    ports:
      - 82:80
    networks:
      - mynetwork

  app_3:
    image: nginxdemos/hello
    ports:
      - 83:80
    networks:
      - mynetwork

  haproxy:
    image: haproxytech/haproxy-alpine:2.4
    ports:
      - 80:80
    volumes:
      - './:/usr/local/etc/haproxy:ro'
    networks:
      -  mynetwork
```

This way, all resources can be addressed directly, but:

* What if we have 15 containers?
* What if we want to easily identify which resource is being tested? 
* What if we want to easily figure out which port is being used?

## Towards a solution

The easiest (and best) solution to tackle this challenge would be to find a way to get each of these containers exposed to the network as if they were their own service, with their own IP and ports.

Although routing into the Docker network, does not seem possible, Docker supports [mavclan networks](https://docs.docker.com/network/macvlan/), which allows a container's NIC to be directly exposed on the Host' network and get its own IP address. Sadly, this feature is not available on Docker desktop. 


### First baby step

But let's not be bothered with port binding (yet), start making our life easier by being able to stop addressing these resources with "localhost" (i.e. `http://localhost:80`, `http://localhost:81`).

By default, docker assigns the container service as a hostname and make it resolvable within the Docker network. In our example, the `haproxy` container can access `app_1` using `http://app_1:80` and since we configured port binding, the host can also access `app_1` but this time only using `http://localhost:81`.

That can be confusing, within the docker network we'd use `http://app_1:80` but we'd use `http://localhost:81` from outside (different hostname, different port).

We're going to slightly change this behavior though, and give each of these containers a [FQDN](https://en.wikipedia.org/wiki/Fully_qualified_domain_name).

```yaml
version: '3.8'

networks:
  mynetwork:
    name: mynetwork

services:
  app_1:
    image: nginxdemos/hello
    hostname: app_1.dev.sandbox.jahia.com
    ports:
      - 81:80
    networks:
      - mynetwork

  app_2:
    image: nginxdemos/hello
    hostname: app_2.dev.sandbox.jahia.com
    ports:
      - 82:80
    networks:
      - mynetwork

  app_3:
    image: nginxdemos/hello
    hostname: app_3.dev.sandbox.jahia.com
    ports:
      - 83:80
    networks:
      - mynetwork

  haproxy:
    image: haproxytech/haproxy-alpine:2.4
    hostname: haproxy.dev.sandbox.jahia.com
    ports:
      - 80:80
    volumes:
      - './:/usr/local/etc/haproxy:ro'
    networks:
      -  mynetwork
```

In this modified `docker-compose.yml` file, the `haproxy` container can access `app_1` using `http://app_1.dev.sandbox.jahia.com:80`, but from the host, we still need to use `http://localhost:81` to access `app_1`.

### DNS to the rescue

From there we can do some trickery, what about creating a DNS zone, with a wildcard `A` record always pointing to `127.0.0.1`.

```
*.dev.sandbox.jahia.com     A       127.0.0.1
```

Using this, any request to a name under `.dev.sandbox.jahia.com` will return the (host's) loopback address `127.0.0.1`

Note: you can also perform something similar using the `/etc/hosts` file on your host and listing all hosts with a resolution to 127.0.0.1.

From that point on:
 - Inside the Docker network, resources can be accessed using their hostname (the DNS wildcard is not used)
 - Outside the Docker network, resources can also be accessed using their hostname, which will redirect the request to the loopback address.

The `haproxy` container can still access `app_1` using `http://app_1.dev.sandbox.jahia.com:80`, and from the host, we can begin using `http://app_1.dev.sandbox.jahia.com:81` to access `app_1`.

Still a different port, but at least, the same hostname.

## HAProxy to the rescue

As you can see, we're already using HAProxy to distribute requests amongst the various container. But could HAProxy help us in this precise scenario ?

And as you can guess the answer is yes.

We're going to update the HAProxy configuration and create one backend by service.

File: `haproxy.cfg`
```
global
  log stdout format raw local0 debug

defaults
  mode http
  timeout connect 30s
  timeout client 60s
  timeout server 600s
  timeout http-request 60s
  log global

frontend fe_main
  bind :80
  use_backend %[req.hdr(Host),lower]

backend app_1.dev.sandbox.jahia.com
  mode http
  server app_1 app_1.dev.sandbox.jahia.com:80 check

backend app_2.dev.sandbox.jahia.com
  mode http
  server app_2 app_2.dev.sandbox.jahia.com:80 check

backend app_3.dev.sandbox.jahia.com
  mode http
  server app_3 app_3.dev.sandbox.jahia.com:80 check

backend haproxy.dev.sandbox.jahia.com
  mode http
  http-response set-header X-Server %s
  server app_1 app_1.dev.sandbox.jahia.com:80 check
  server app_2 app_2.dev.sandbox.jahia.com:80 check
  server app_3 app_3.dev.sandbox.jahia.com:80 check
```

Using this configuration, the backend is defined by the FQDN used when making the request (see: `use_backend %[req.hdr(Host),lower]`). 

This means that an HTTP request to `app_1.dev.sandbox.jahia.com` received by HAProxy will be forwarded to `app_1.dev.sandbox.jahia.com`. But an HTTP request to `haproxy.dev.sandbox.jahia.com` will be forwarded to any for the `app_` server.

And remember, the host would be using the DNS wildcard configured about, so from the host, a request to `app_1.dev.sandbox.jahia.com:80` will translate to `127.0.0.1:80` and a request to `haproxy.dev.sandbox.jahia.com:80` will also translate to `127.0.0.1:80`.

As a result we actually only need to expose one single port on the haproxy container, and let that container dispatch the request based on its configuration.

File: `docker-compose.yml`
```yaml
version: '3.8'

networks:
  mynetwork:
    name: mynetwork

services:
  app_1:
    image: nginxdemos/hello
    hostname: app_1.dev.sandbox.jahia.com
    networks:
      - mynetwork

  app_2:
    image: nginxdemos/hello
    hostname: app_2.dev.sandbox.jahia.com
    networks:
      - mynetwork

  app_3:
    image: nginxdemos/hello
    hostname: app_3.dev.sandbox.jahia.com
    networks:
      - mynetwork

  haproxy:
    image: haproxytech/haproxy-alpine:2.4
    hostname: haproxy.dev.sandbox.jahia.com
    ports:
      - "80:80"
    volumes:
      - './:/usr/local/etc/haproxy:ro'
    networks:
      -  mynetwork

```

## Conclusion

At the beginning of this article, we had to rely on port binding for each container we wanted to access from the host.

Going from:

| Inside Docker | Outside docker |
| --- | --- |
| HAProxy:80 | localhost:80 |
| APP_1:80 | localhost:81 |
| APP_2:80 | localhost:82 |
| APP_3:80 | localhost:83|

To:

| Inside Docker | Outside docker |
| --- | --- |
| haproxy.dev.sandbox.jahia.com:80 | haproxy.dev.sandbox.jahia.com:80 |
| app_1.dev.sandbox.jahia.com:80 | app_1.dev.sandbox.jahia.com:80 |
| app_2.dev.sandbox.jahia.com:80 | app_2.dev.sandbox.jahia.com:80 |
| app_3.dev.sandbox.jahia.com:80 | app_3.dev.sandbox.jahia.com:80|

This is not an ideal solution, it definitely adds complexity and introduce a new layer/breaking point. Is that worth it?

## What's next?

The above configuration is limited to HTTP, but it seems that HAProxy can support a similar configuration for other protocols, such as SSH: https://www.haproxy.com/blog/route-ssh-connections-with-haproxy/


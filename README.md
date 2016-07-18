# Context
I've started to use docker at work and I have a personal server that hosts 2
websites on a single nginx instance, with 2 virtual hosts configured (a.k.a. server). So the idea is to upgrade the architecture
to embrace the docker way, and also improve my knowledge about docker. 

Each website should run on its own container, and a reverse proxy will route the traffic.

# Implementation

## Step1
Let's create a new project, divided in 3 folders : site1, site2, proxy. One folder for each container.

	proxy
	  Dockerfile
	  conf
	    conf.d
	      site1.conf
	      site2.conf
	site1
	  Dockerfile
	  index.html
	site2
	  Dockerfile
	  index.html

Here is the content of `site1.conf`, the reverse proxy configuration for site1 :

    server {
        listen       80;
        server_name  site1;

        location / {
            proxy_pass   http://192.168.59.103:8080/;
        }
    }

Let's run the containers :

	docker run -d -p 8080:80 ymn/site1
	docker run -d -p 8081:80 ymn/site2
	docker run -d -p 80:80 ymn/proxy

I can test direct access

	curl "http://docker:8080/"
	curl "http://docker:8081/"

and proxified access

	curl "http://site1"
	curl "http://site2"

What could be improved :

* *proxy_pass* contains an hardcoded IP adress
* The 3 containers should be started manually
* The ports of site container are publicly exposed

## Step2

To avoid exposing ports, Docker allows to [link containers together](http://docs.docker.com/userguide/dockerlinks/).

Here is how to run containers

    docker run -d --name site1 ymn/site1
    docker run -d --name site2 ymn/site2
    docker run -d -p 80:80 --name proxy --link site1 --link site2 ymn/proxy

As you can see, containers now have names, because Docker linking is 
based on the name.

Here is the new configuration for the proxy

    server {
        listen       80;
        server_name  site1;

        location / {
            proxy_pass   http://site1/;
        }
    }

With the link, the proxy container know how to connect to site1. In fact,
Docker adds a host entry for site1 to the `/etc/hosts` of the proxy container.

## Step3

Command lines are now very long and it becomes painfull to run the whole things.
Hopefully, there is a tool in the fast-growing Docker ecosystem that will help
us : [docker-compose](https://www.docker.com/docker-compose).

We just have to define our multi-container application with a configuration file in YAML to describe how to run our 3 containers

    proxy:
      build: proxy
      ports:
      - 80:80
      links:
      - site1
      - site2
    site1:
      image: ymn/site1
    site2:
      image: ymn/site2

And we can run all containers with a single command : `docker-compose up`

## Step 4

The setup is now quite good. But there is still way to improve. Docker expose a [remote API](http://docs.docker.com/reference/api/docker_remote_api/) that allows to automate many tasks. So some projects leverage this API, like [nginx-proxy](https://github.com/jwilder/nginx-proxy) which  generate automatically the config of the proxy:

    docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro jwilder/nginx-proxy
    docker run -d -e VIRTUAL_HOST=site1 ymn/site1
    docker run -d -e VIRTUAL_HOST=site2 ymn/site2

# Links

* http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/
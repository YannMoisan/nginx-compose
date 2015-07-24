# Context
I've started to use docker at work and I have a personal server that hosts 2
websites on a single nginx instance. So the idea is to upgrade the architecture
to embrace the docker way, and by this way improve my knowledge about docker. 

Each website should run on its own container, and a reverse proxy should route the traffic.

# Implementation
## Step1
The project is divided in 3 folders : site1, site2, proxy. One folder for each container.

Here is the nginx config for the proxy

    server_name  site1;

    location / {
        proxy_pass   http://192.168.59.103:8080/;
    }

I need to run containers :

	docker run -d -p 8080:80 ymn/site1
	docker run -d -p 8081:80 ymn/site2
	docker run -d -p 80:80 ymn/proxy

Direct access

	curl "http://docker:8080/"
	curl "http://docker:8081/"

With the proxy

	curl "http://site1"
	curl "http://site2"

What could be improved :

* proxy_pass contains an hardcoded IP adress
* The 3 containers should be started manually
* port of site container are publicly exposed

# Links

* http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/
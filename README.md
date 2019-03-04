# nginx-lua-proxy - dockerized Nginx+Lua dynamic proxy with redis store for backends


![Docker stars](https://img.shields.io/docker/stars/ermlab/nginx-lua-proxy.png "Docker stars")
&nbsp;
![Docker pulls](https://img.shields.io/docker/pulls/ermlab/nginx-lua-proxy.png "Docker pulls")

[![Docker repo](https://github.com/Ermlab/nginx-lua-proxy/blob/master/images/docker.png?raw=true "Docker repo")](https://hub.docker.com/r/ermlab/nginx-lua-proxy/)
&nbsp;
[![Github repo](https://github.com/Ermlab/nginx-lua-proxy/blob/master/images/github.png?raw=true "Github repo")](https://github.com/Ermlab/nginx-lua-proxy)

The main goal is to build the counterpart of hipache (https://github.com/hipache/hipache) with nginx.
The proxy tries to find the host in redis database and without the reloading (proxy server) use it as upstream server.

The data stored in redis is in the same format as in hipache. All code is built from source.

This procject is based on wonderfull projects:
* https://github.com/danday74/docker-nginx-lua
* https://github.com/samalba/hipache-nginx

## Usage


1. Run REDIS database, it is essential to name it **redis**, because the lua resty redis connection objects relies on hostname=redis,

    ```
    docker run -d --name redis redis
    ```

2. Run nginx-lua-proxy container and linked it with redis

    ```
    docker run -d --link redis:redis -p 9090:80 --name $CONTAINER_NAME ermlab/nginx-lua-proxy
    ```


3. Add to redis some hosts
   ```
   $ redis-cli rpush frontend:dynamic1.example.com mywebsite
   $ redis-cli rpush frontend:dynamic1.example.com http://192.168.0.50:80

   $ redis-cli rpush frontend:dynamic2.example.com mywebsite
   $ redis-cli rpush frontend:dynamic2.example.com http://192.168.0.100:80
   ```

4. Check if everything is working

   ```
   curl -H 'Host: dynamic1.example.com' http://localhost:9090

   or

   curl -H 'Host: dynamic2.example.com' http://localhost:9090
   ```

5. If you want to test in the browser you should set the dns wildcard for domain \*.example.com and it should point to your nginx proxy

## Performance testing Hipache vs NGINX

Testing scenario:
* at front sits haproxy and do routing between two backends: hipache.ermlab.com and nginx.ermlab.com
* haproxy redirects traffic from \*.hipache.ermlab.com to hipache proxy and \*.nginx.ermlab.com to nginx-lua-proxy
* haproxy, hipache, nginx-lua-proxy and redis are installed on the same server (proxy server)
* there is one simple static website, it is available at 192.168.0.10  (web server)
* redis contains two dynamic backends both point to the same website (192.168.0.10)
    * host for hipache: id1.hipache.ermlab.com->192.168.0.10
    * host for nginx-lua-proxy: id1.nginx.ermlab.com->192.168.0.10
* software runs as docker containers: redis, hipache, nginx-lua-proxy
* proxy server and web server have 2CPUs and 2GB RAM

Testing with apache benchmark

```
ab -n 20000 -c 200 http://id1.hipache.ermlab.com
ab -n 20000 -c 200 http://id1.nginx.ermlab.com
```

### Results



Parameter  | Hipache | Nginx-lua-proxy
-------------: | :-------------|:----------
Concurrency Level:     | 200 | 200
*Time taken for tests:  | 57.446 seconds | 14.951 seconds
Complete requests:     | 20000 | 20000  
Failed requests:       | 0 | 0
Write errors:          | 0 | 0
Total transferred:     | 6500000 bytes | 6380000 bytes
HTML transferred:      |2680000 bytes | 2560000 bytes
**Requests per second:   | **348.15 \[#/sec\] (mean)** | **1337.68 \[#/sec\] (mean)**
*Time per request:      | 348.464 [ms] (mean) | 149.513 [ms] (mean)
*Time per request:      | 2.872 [ms] | 0.748 [ms]
**Transfer rate:         | 110.50 [Kbytes/sec] | 416.65 [Kbytes/sec]

*Lower is better

**Higher is better 

Our solution is 3-4x better than Hipache.

### Hipache - connection times



Connection Times (ms) |  min | mean |[+/-sd] | median |  max
------------|------|---|------|-------|---------------            
Connect:    |   0  |20 | 362.2 |      1  |  7001
Processing: |    4 | 456 | 653.2 |   398 |   15349
Waiting:    |    3 | 453 | 653.3 |    395 |   15349
Total:      |    5 | 477 | 744.6 |    400 |  15350



### Nginx-lua-proxy - connection times


 Connection Times (ms) |  min | mean |[+/-sd] | median |  max
 ------------|------|---|------|-------|---------------   
 Connect:     |   0 |   1 |   0.5 |     1 |     16
 Processing:  |  40 | 143 | 197.9 |   110 |   3297
 Waiting:     |  40 | 143 | 197.9 |   110 |   3297
 Total:       |  46 | 144 | 197.9 |   111 |   3298


### Percentage of the requests served within a certain time(ms) - lower is better

|Percent | Hipache (ms) | Nginx-lua-proxy (ms) |
|----|------|-------------- |
|50% |  400 |  111 |
|66% |  484 |  120 |
|75% |  546 |  126 |
|80% |  584 |  129 |
|90% |  687 |  138 |
|95% |  794 |  152 |
|98% |  897 | 198 |
|99% | 1032 | 515 |
|100% (longest request) |15350 |  3298 |


## VHOST Configuration


All VHOST configuration is managed through a REDIS. This makes it possible to update the configuration
dynamically and gracefully while the server is running, and have that state
shared across workers.

Let's take an example to proxify requests to 2 backends for the hostname
`example.com`. The 2 backends IP are `192.168.0.42` and `192.168.0.43` and
they serve the HTTP traffic on the port `80`.

`redis-cli` is the standard client tool to talk to Redis from the terminal.

Follow these steps:

1. __Create__ the frontend and associate an identifier:

        $ redis-cli rpush frontend:example.com mywebsite
        (integer) 1

The frontend identifer is `mywebsite`, it could be anything.

2. __Associate__ the 2 backends:

        $ redis-cli rpush frontend:example.com http://192.168.0.42:80
        (integer) 2
        $ redis-cli rpush frontend:example.com http://192.168.0.43:80
        (integer) 3

3. __Review__ the configuration:

        $ redis-cli lrange frontend:example.com 0 -1
        1) "mywebsite"
        2) "http://192.168.0.42:80"
        3) "http://192.168.0.43:80"

While the server is running, any of these steps can be re-run without messing up
with the traffic.

## Automated


The master branch on the github repo is watched by an automated docker build

Which builds docker image **ermlab/nginx-lua** on push to master

On success, the docker build triggers the docker repo's webhooks (if any)

## Maintainers

* [Ermlab software house](http://ermlab.com)
    * [Krzysztof Sopyła](https://github.com/ksopyla) (sopyla@ermlab.com)

## License


[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

APACHE LICENSE-2.0 ... In other words, please use freely and do whatever you want with it for good of all people :)
